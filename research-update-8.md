# Week Of 1/14 Research Update
This week was a rather successfully week, as I was able to both configure my development environment properly and I finally got around to implementing what I would call "v1" of my plan to insert no-op's into binary files. Last week I had a goal of devising an object-oriented plan to alter return statements. It dawned upon me that all of LLVM is basically an OOP structure, so I could just use the objects/structures already in place and add functionality to them (in the appropriate classes of course). 

LLVM is indeed a complex OOP program...

<p align="center">
  <img src ="https://i.imgur.com/Pf7seDe.png" />
</p>


## A Devops Triumph
This week I finally got frustrated with not having a debugger. Early on in the lifecycle of this project when I set up my Visual Studio LLVM development environment, I noticed my debugger wasn't working. I spent a few hours trying to figure it out before eventually just deciding to live without it. This week, having finally started working with the backend of LLVM, I decided it was time to figure out what was happening. No longer could I live with just string outputs for debugging purposes.

It took me about 2 days, which I'm sort of ashamed of because the solution seems so obvious now. First, I should mention that we spent a lot of time talking about debug formats, PDB files, and debug functionality at PlayStation when discussing out compiler. This is mainly because pretty much every AAA game released on the market relies heavily on optimizations to stay in a runnable state, which in turn makes debugging a bit more difficult. Thus, I'm not new to the way clang handles debug files...but I apparently forgot.

It wasn't until I found the following page of [documentation](https://clang.llvm.org/docs/MSVCCompatibility.html) that I was reminded of what to do.
```"Clang emits relatively complete CodeView debug information if /Z7 or /Zi is passed. Microsoft’s link.exe will transform the CodeView debug information into a PDB that works in Windows debuggers and other tools that consume PDB files like ETW"```

Basically, I needed to compile each project with the `/Z7` flag to produce PDB files that work with Visual Studio. I should have known this. After making this change, things still didn't work. After much confusion, I remembered how the clang driver works. The clang driver works by spawning multiple processes (one for compiling, linking, etc). This was [documented](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-11-5-update.md) early on in the lifecycle of the project. So I realized that when I clicked "run" in Visual Studio, my breakpoints weren't being hit because they were being executed on a background thread (all my breakpoints were in the backend portion of our compilation process). Luckily, Microsoft had a [solution](https://marketplace.visualstudio.com/items?itemName=vsdbgplat.MicrosoftChildProcessDebuggingPowerTool) ready to go. The Microsoft Child Process Debugging Power Tool.

<p align="center">
  <img src ="https://i.imgur.com/YCmmDqL.png" />
</p>

After wrestling with my development environment, I was able to implement my first solution to the ROP-gadget problem.


## Noops
Remember, the goal I've set is to devise a solution to make ROP gadgets harder to find. One potential solution is to insert Noop instructions before return instructions, thus separating the usefulness (the instruction before the return) from the actual return. Ideally, the basic tools people use to find gadgets would have trouble detecting the gadgets at that point. It's about fooling the tools people use to find exploits. Last week, I spent a considerable amount of time investigating the `LowerReturn()` function of the x86 backend generator. After reading the following [page](http://llvm.org/docs/ProgrammersManual.html#creating-and-inserting-new-instructions) of LLVM documentation, I realized that might not be the best place to alter the stream of instructions. The problem is that when FastInstructionSelection is enabled (which it usually is), the lowering process doesn't even take place. It occurred to me after reading the LLVM doc that altering the basic block might be the best approach: the basic blocks contain the "stream" of machine instructions we can alter. This lead me to a discussion with one of the compiler engineers at Sony, who pointed me to the following [Phabricator Review](https://reviews.llvm.org/D6983) for a similar approach. The problem is that this person added the functionality to add no-ops to EVERY instruction in an executable. I don't want to do that, I want to only focus on return instructions. I did appreciate their approach of altering the stream of instructions within the LLVM BasicBlock's by using the `BuildMI()` function which allows you to append machine instructions together, something I incorporated in my own revision.

From here, I decided it was time to host my own fork of LLVM which you can find [here](https://github.com/justiceadamsUNI/llvm). I forked LLVM not clang because we are dealing with backend alterations, not front end (Clang). The branch [Rop-research-base](https://github.com/justiceadamsUNI/llvm/tree/ROP-research-base) corresponds to the SVN revision 346633 @ trunk which we decided on way back in [November](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-11-5-update.md). I branched from the base and created branch ROP-Noop-Insertion-V1 where the following commit lives

https://github.com/justiceadamsUNI/llvm/commit/6165888f28267cdc10b1d6bb878f5f1e2716b0ce

This is the first version of our no-op insertion. At the SelectionDagLevel, after everything is lowered and adjusted properly, we now insert a no-op before every return instruction. Consider the following simple C++ code

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

Consider the assembly of the `foo()` function BEFORE my no-op insertion change. You would see this in the binary
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
|           0x140001024      4883c410       add rsp, 0x10
\           0x140001028      c3             ret
```

AFTER the no-op insertion, you see this:
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
Thus, we have seperated the `idiv` instruction from the ret instruction with a noop successfully. You'll notice that the stack pointer is still being adjusted. I would like another noop between that and the actual `ret`, but they are intrinsically tied together. So much so, that it's very hard to actually seperate them, but it is on my "Investigate more" list.

Thus, this is the alpha stage of what will become **hopefully** the solution. Now that we're here, I'll outline my goals going forward.

Note that I used the radare2 tool I've been using all year to see the binary output of each function above.


## For Next Week(s)
- I would like to add a random number of no-ops before the return. Also, I would like to explore the possibility of returning other logical no-ops. What I mean by logical noop is an instruction that has no effect. Moving a register to itself, for example, Adding 0 to an integer, things like that. This way the no-ops can "hide" properly in the assembly. This would require a rewrite of `getNoop()` in the X86 implementation of `TargetInstrInfo` 
- Figure out if there is a way to seperate the actual final instruction (usually this is popping the stack or adding to the stack pointer) from the `ret` instruction. This is tough, i tried, but they are so intimately tied together in the instruction stream. They are considered one MachineInstruction (that's a note for myself). It needs more investigation.
- Once we get a "suitable" prototype, I would like to compile some larger binaries and use some gadget-finding tools to see if the results are actually beneficial. I really don't know right now... I guess that's why it's called research.

## Sources
- [Clang Documentation - MSVC compatibility](https://clang.llvm.org/docs/MSVCCompatibility.html)
- [Microsoft Child Process Debugging Power Tool](https://marketplace.visualstudio.com/items?itemName=vsdbgplat.MicrosoftChildProcessDebuggingPowerTool)
- [LLVm Programmers Manual: Creating New Instructions](http://llvm.org/docs/ProgrammersManual.html#creating-and-inserting-new-instructions)
- [Instruction Class Reference](http://llvm.org/doxygen/classllvm_1_1Instruction.html)
- [NoOp Insertions Before Every Instruction](https://reviews.llvm.org/D6983)
- [MachineInstrBuilder Source](http://llvm.org/doxygen/MachineInstrBuilder_8h_source.html)
- [X86ISelLowering.cpp Source](http://llvm.org/doxygen/X86ISelLowering_8cpp_source.html)
- [Inserting nodes into SelectionDAG (X86)](https://groups.google.com/forum/#!topic/llvm-dev/6L2Wfeh5K_A)

## Image Sources
- http://llvm.org/doxygen/classllvm_1_1SelectionDAGISel.html
- https://clang.llvm.org/docs/MSVCCompatibility.html
