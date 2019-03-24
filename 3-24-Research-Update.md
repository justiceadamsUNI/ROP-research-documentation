# Week Of 3/24 Research Update
Over the past few weeks I've been evaluating my modified [v3](https://github.com/justiceadamsUNI/llvm/tree/ROP-Noop-Insertion-v3) compiler and measuring the number of gadgets found in applications. It wasn't very fruitfull however, as I was seeing very small decreases in gadget detection numbers. I took a dive into the gadget detectors output and noticed that certain gadgets were nowhere to be found using objdectdump. I decided to go back to the drawing board, compile an incredibly small C++ application to see what gadgets were being listed, and work my way up. Along the way I adjusted the compiler once again to actually solve one of the major problems outlined way back in [January](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-1-14-Research-Update.md).

# V4
So I noticed that many of the gadgets being listed were legitmate instructions that appeared right before the return instruction. If you recall from [January](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-1-14-Research-Update.md), one of the problems I discovered was not beiong able to seperate the final returning cleanup instruction from the actual return. I could add nops before the return instruction, but it seemed like the return instruction was intriniscly tied to the machine instruction right before it... and I coudn't seem to figure out why. Take this example I outlined way back in January.

Let's compile the simple c++ code with only one nop insertion using our v3 compiler. 
```
int foo(int a, int b, int c) {
  int sum = a + b;
  return sum / c;
}

int main() 
{
	return foo(10,10,10);
}
```

What we would see is this in the assembly code,
```
|           0x140001000      4883ec10       sub rsp, 0x10              ; [00] m-r-x section size 44544 named .text
|           0x140001004      448944240c     mov dword [local_ch], r8d
|           0x140001009      89542408       mov dword [local_8h], edx
|           0x14000100d      894c2404       mov dword [local_4h], ecx
|           0x140001011      8b4c2404       mov ecx, dword [local_4h]  ; [0x4:4]=-1 ; 4
|           0x140001015      034c2408       add ecx, dword [local_8h]
|           0x140001019      890c24         mov dword [rsp], ecx
|           0x14000101c      8b0424         mov eax, dword [rsp]
|           0x14000101f      99             cdq
|           0x140001020      f77c240c       idiv dword [local_ch]
|           0x140001024      90             nop         <-------- NOTICE THE NOP INSTRUCTION
|           0x140001025      4883c410       add rsp, 0x10
\           0x140001029      c3             ret
```
Notice that there is still the `add rsp, 0x10` instruction right before the `ret`. I couldn't find a way around this, and it was contributing to the overall gadget count. V4 fixes this issue by moving all nop insertion into a file called `x86FrameLowering.cpp`. This particular file handles the implementation for setting up stack frames on an x86 architecture. It handles setting up and cleaning up stack frames, but does so at the selectionDAG level, which as you may recall is where we [earlier decided](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-1-7-Research-Update.md) was the best place to handle program alteration (nop insertion). I've pushed v4 of the compiler [here](https://github.com/justiceadamsUNI/llvm/commit/53a8b6b38ac87e5b7ddb5b9aede4ce9e8a80db4c). I managed to move the functionality into the frame lowering portion of the code generation while leaving the old implementation for logical NoOps (which has been abandoned for the time being. More on that in a bit). Now the compiled code would look something like this.

```
|           0x140001000      4883ec10       sub rsp, 0x10              ; [00] m-r-x section size 44544 named .text
|           0x140001004      448944240c     mov dword [local_ch], r8d
|           0x140001009      89542408       mov dword [local_8h], edx
|           0x14000100d      894c2404       mov dword [local_4h], ecx
|           0x140001011      8b4c2404       mov ecx, dword [local_4h]  ; [0x4:4]=-1 ; 4
|           0x140001015      034c2408       add ecx, dword [local_8h]
|           0x140001019      890c24         mov dword [rsp], ecx
|           0x14000101c      8b0424         mov eax, dword [rsp]
|           0x14000101f      99             cdq
|           0x140001020      f77c240c       idiv dword [local_ch]
|           0x140001025      4883c410       add rsp, 0x10
|           0x140001026      90             nop         
|           0x140001027      90             nop         
|           0x140001028      90             nop         
|           0x140001029      90             nop         
|           0x14000102a      90             nop         
|           0x14000102b      90             nop         
|           0x14000102c      90             nop         
|           0x14000102d      90             nop         
|           0x14000102e      90             nop         
|           0x14000102f      90             nop         
\           0x140001030      c3             ret
```

Which should make things better due to the seperation between the `add rsp` command and the actual return instruction. A small win indeed.

# Ignoring libraries
Part of the problem I was running into when looking at the output of my dadget finders tool was that I was seeing gadgets reported in functions I had never written. Where on earth were these gadgets coming from? Well, it's rather simple really. I'm actually a bit* ashamed it took me so long to find this, but they're coming from both the C++ runtime or the Standard Library (or possible another library dependeing on the program itself being compiled). I realized then, that I really shouldn't be examining entire linked together binary applications. Instead, I should just focus on the pre-linked object file output being that this is where I would be able to actually see results when evaluating my compiler. I can't prevent gadgets in any libraries unless I recompile those libraries with my modified compiler, and for the scope of this project I *probably* don't plan on recompiling the entire standard library. That would be a stretch goal if anything, but for now I'm simply going to go forth analyzing compilation output only by passing the [`-c` command](https://clang.llvm.org/docs/ClangCommandLineReference.html) to the clang driver, telling it to only run preprocess, compile, and assemble steps. This should give me a better idea about the efficiency of the modified compiler.

# Testing on a small scale
I started with testing a very small application. Take the following c++ application:
```
int main() {
	int x = 60 + 50 - 20;
	return -100;
}
```
It's nothing special, and doesn't make much sense, but it is valid C++. Using the original version of clang and compiling this program, the gadget finder will detect 6 gadgets after analyzing the program. The output is below
```
Gadgets information
============================================================
0x0000000000000113 : add byte ptr [rax - 0x64], bh ; pop rcx ; ret
0x0000000000000111 : add byte ptr [rax], al ; add byte ptr [rax - 0x64], bh ; pop rcx ; ret
0x0000000000000112 : add byte ptr [rax], al ; mov eax, 0xffffff9c ; pop rcx ; ret
0x0000000000000114 : mov eax, 0xffffff9c ; pop rcx ; ret
0x0000000000000119 : pop rcx ; ret
0x000000000000011a : ret

Unique gadgets found: 6
```

Now let's look at our v4 compiler output. When analyzing the gadget output of this compiled code, the tool reports 10 gadgets. We must have made it worse right? Wrong, we made it much better!
```
Gadgets information
============================================================
0x000000000000011b : nop ; nop ; nop ; nop ; nop ; nop ; nop ; nop ; nop ; ret
0x000000000000011c : nop ; nop ; nop ; nop ; nop ; nop ; nop ; nop ; ret
0x000000000000011d : nop ; nop ; nop ; nop ; nop ; nop ; nop ; ret
0x000000000000011e : nop ; nop ; nop ; nop ; nop ; nop ; ret
0x000000000000011f : nop ; nop ; nop ; nop ; nop ; ret
0x0000000000000120 : nop ; nop ; nop ; nop ; ret
0x0000000000000121 : nop ; nop ; nop ; ret
0x0000000000000122 : nop ; nop ; ret
0x0000000000000123 : nop ; ret
0x0000000000000124 : ret
```
You can see that the "gadgets" found aren't really gadgets at all. None of these "gadgets" provides any functionality to an attacker trying to generate a ROP chain. If you were to execute any of the gadgets listed, you wouldn't be doing anything at all besides executing `nop` instuctions on the cpu.  Thus, not only have we hidden all the gadgets previously reported, we actually managed to generate a bunch of false positives, thus making the attackers life much harder. This is due to the way the gadget tools are written. They aren't executing any of the instructions, so what they do is simply report based off the condition "this instruction is before a return, therefore it's a gadget". This is a very big win.

Now let's look at a more complicated file, the [Parser.cpp](https://github.com/justiceadamsUNI/Klein-Compiler/blob/master/Compiler/src/implementation/Parser.cpp) of my Klein compiler project. When looking at unique ROP gadgets found compiling with default clang, we find 3586 gadgets. When compiling with our v4 compiler we find 2851 gadgets.

That's a 20% decrease in gadgets for what is only a 1% increase in object file size. That's pretty cool, but more testing needs to be done, and some more safegaurds need to be put in place for this approach. Also I only counted unique gadgets, The **total** gadget differential is 14531 gadgets when using the standard compiler to a 5721 gadget count. That's roughly a 60% decrease in total gadget counts, although this metric includes duplicates so it might not be as valuable.

# Abandoning logical noops
You may have noticed that the current v4 compiler is using strict `nop` instructions. This is because I ran into an issue when evaluating logical nops. Let's take one of the instructions I was using as an example. Copying the `%rsp` register into itself. While this is indeed a "logical nop" as it doesn't change the logical validity of our compiled program, it introduced a new problem. New gadgets were being detected using my logical nop instructions. How? Well I think it's because certain instructions take up more than one byte in memory. If an instruction takes up 4 bytes in memory, you could theoretically start execution at the second byte, and in x86 architecture, this could very well be a completely different instruction, and thus introduce a new gadget. I haven't dug into this enough yet, but I'm planning on asking some of the compiler engineers at PlayStation on monday about this very topic. This is something I would like to introduce in the future, but might require carefull selection of instructions. We need an instruction(s) that themselves are nops, but also can't be spliced into other usefull instructions... this might be quite the challenge.

# Moving forward
Here are the things I would like to focus on in the last month or so of the project.

1.) We now have a functioning clang compiler which can reduce gadget detection quite reliably. The problem is that we are using regular `nop` instructions right now. I would like to alter this to include logical nops, because my fear is that you could simply update the gadget finders to ignore nop instructions and thus gadgets would be detectable once more. This will require some extenxive research into what instructions won't introduce more gadgets into the resulting object code.

2.) Circle back to Unreal Engine and once again drop in the modified compiler, this time examining only object code compiled from the compiler. I'm fairly certain this is possible, but I really don't know. I should have thought about this while at GDC so I could ask the Epic Games people...

3.) Develop an automated system for measuring compiler output. It would be nice to set up a TravisCI job or somehting that I could just hit run and it would report the effectivness of our compiles gadget hiding similar to what I postrd above. It's becoming a pain to have to run the gadget finders with each small change and manually write down the differences. Why not automate this process for easier evaluation (if there's time).

4.) Spend some time thinking and documenting what it would actually take to move this change upstream. What would need to be done before submitting a phabriacator review to the LLVM community.

# Information Sources
- [HexagonFrameLowering class reference](http://llvm.org/doxygen/classllvm_1_1HexagonFrameLowering.html)
- [ROP Gadget Finder Tool](https://github.com/JonathanSalwan/ROPgadget)
- [Clang FAQ](https://clang.llvm.org/docs/FAQ.html)
- [LibTooling documentation](https://clang.llvm.org/docs/LibTooling.html#libtooling-builtin-includes)
- [Clang Command Line Reference](https://clang.llvm.org/docs/ClangCommandLineReference.html)
- [RaDare2 documentation](https://github.com/radare/radare2/blob/master/doc/intro.md)
- [Memory Address Wikipedia](https://en.wikipedia.org/wiki/Memory_address)
- [Hexidecimal Compute Table](https://en.wikipedia.org/wiki/Hexadecimal)
