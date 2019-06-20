# Week(s) Of 2/4 to 2/18 Research Update
Over the last few weeks, I've been evaluating my modified compilers success rate in preventing ROP gadget detection. I used some Unreal Engine examples as metrics which I quickly realized isn't a super accurate approach. From there, I started tinkering with the No-op insertion from [V2](https://github.com/justiceadamsUNI/llvm/tree/ROP-Noop-Insertion-V2) in order to create a more successful version of the ROP-safe compiler and gathered some metrics from compiling my old Klein compiler.

# Too Many Cooks In The Kitchen
I spent some time plugging my compiler into Unreal Engine and evaluating the resulting binaries. In almost every run, I found that the ROP gadgets found according to the following [open source gadget finder](https://github.com/JonathanSalwan/ROPgadget) using the v2 compiler only reduced gadgets found by approximately ~1-2%. In part, this is because of the sheer number of elements being combined to build both the object code and the resulting binary using Unreal. This is something I probably should have seen coming. When building a game package, you have to deal with the regular C++ compiler, the [Unreal header tool](https://ericlemes.com/2018/11/23/understanding-unreal-build-tool/), precompiled headers, precompiled libraries that ship with Unreal Engine, [The shader compiler](https://docs.unrealengine.com/en-us/Programming/Rendering/ShaderDevelopment#shadercompilation), and the [Blueprint Compiler](https://docs.unrealengine.com/en-us/Engine/Blueprints/TechnicalGuide/Compiler). Anyone of these components can ultimately contribute to the resulting executable which in turn makes it hard to determine just how much "safer" the resulting binary is. In each scenario, I found that the resulting binary only reported about ~1-2% fewer gadgets. This, in turn, revealed that my v2 compiler wasn't working super well.

![Imgur](https://i.imgur.com/wWpuBd9.png)


# The Klein Compiler Makes a comeback
After realizing that the Unreal approach wasn't necessarily an accurate representation (for right now), I decided I needed to use some sort of C++ source to compile and evaluate. I wanted the codebase to be big enough that the metric is useful, but I didn't want it to be so big that it would take forever to compile (at first I was going to build clang using my modified clang, but it's too big). At this point, I remembered my [Klein compiler](https://github.com/justiceadamsUNI/Klein-Compiler). It's just about the right size having reported 6502 gadgets using a non-modified version of clang. My goal is to use this (and probably some other smaller projects) to iterate on my compiler until I can get at least a 10% deduction in gadgets found, at that point, I will go back to using Unreal Engine and find out just how much this could impact a resulting game. I'm hoping for rougly 3-5% decrease in the overall game package.


# V3 approaches
After evaluating the Klein compiler with my v2 no-op insertion it became clear that the Gadget finder has an easy time figuring out chains of `nop` instructions. So much so that it would report gadgets that were similar to the following 
```nop; nop; nop; nop; pop %rsi%; ret;```

The finder has an easy time just working around a chain of `nops`. What this means is that I needed to come up with some other mechanism besides an actual `nop`. I've added a method called `insertLogicalNop()` to `X86InstrInfo.cpp` which I think could take care of this. What it does, is either add a `nop` instruction OR adds a copy instruction where it copies either the `RSP` or `RBP` register back into itself. These instructions don't do anythin, but they add instructions into our code. These registers were chosen because they are RARELY ever in use outside the context of allocating/popping stack frames. This means we don't have to worry about clang erroring because the register is allready in use (I basically found this out by brute forcing through all the registers, and refering to [this](http://6.035.scripts.mit.edu/sp17/x86-64-architecture-guide.html) document). So now, we have our "Logical nops": A term I'm going to use going forward. Using this approach, I've found that I can decrease gadget detection by about 10% in the Klein compiler, but I need to add about 20 instructions before each return instruction to really jumble the instruction stream. 10% was my goal, and it just turns out that about 20 instructions is what you need to make that work. Unfortunately, it adds about 28% to the resulting executable size. I'm going to investigate this further and see what I can do to lessen the impact of the executable inflation (if at all). The v3 branch of the compiler can be found here: https://github.com/justiceadamsUNI/llvm/tree/ROP-Noop-Insertion-v3

# For Next Week(s)
- Clean up this repo with a better naming scheme for files. The dates are not easy to sift through anymore now that it's a new year. I done goofed.
- Figure out a way to minimize executable inflation (if one exist) and continue iterating on the modified compiler to decrease gadget detection
- Go back to Unreal Engine source and see if we can make a meaningful dent in the number of gadgets found in the resulting assembly code.


# Sources
- [Understanding Unreal Build Tool](https://ericlemes.com/2018/11/23/understanding-unreal-build-tool/)
- [Unreal Engine Shader Development](https://docs.unrealengine.com/en-us/Programming/Rendering/ShaderDevelopment)
- [Unreal Engine Build Tools](https://docs.unrealengine.com/en-us/Programming/BuildTools) 
- [Unreal Engine Blueprint Compiler Overview](https://docs.unrealengine.com/en-us/Engine/Blueprints/TechnicalGuide/Compiler)
- [ROP Gadget Finder](https://github.com/JonathanSalwan/ROPgadget)
- [X86InstrBuilder.h file Reference](https://github.com/JonathanSalwan/ROPgadget)
- [X86-64 Architecture Guide](http://6.035.scripts.mit.edu/sp17/x86-64-architecture-guide.html)
