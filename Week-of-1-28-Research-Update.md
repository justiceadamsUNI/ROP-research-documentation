# Week Of 1/14 Research Update
Over the past two weeks, I have managed to add a level of randomness to my Noop insertions, I plugged my custom compiler into Unreal Engine, and outlined what future work needs to happen before the year's end. Most of my time was spent figuring out a way to use my custom compiler with Unreal Engine, which turns out, is not easy.

## No-ops Version 2
One of the things I wanted to do as part of my no-op insertion into binaries, is to add a certain level of randomness to the binaries. Rather than simply adding a static number of noop instructions each time, I wanted to insert a random number before each return instruction. This ensures that people can't simply predict the pattern with each return instruction and somehow work around it. Attackers could theoretically update their gadget finding tools to recognize the pattern of common loops (for instance if we just added 2 noops before each return). For now, I've took the approach of adding a random number of noops between 1 and 10. In this way, there is a certain level of uncertainty when analyzing any given return instruction in the assembly code. That commit lives [here](https://github.com/justiceadamsUNI/llvm/commit/3a8e60c7ddb4bd8c572493fc62077de5ed2ff176). 

The next step is to add Logical nopps to make patterns even harder to find. A "logical noop" (a term I just made up) is an instruction that does nothing, but may not actually be a nop instruction, which in our case (x86-64) is an actual instruction. Adding 0 to a register, for example, could be considered a "Logical Noop". If we could utilize these dummy instructions, it may help make gadgets even harder to detect. This is my current idea for improving gadget hiding.

# Unreal Engine
Ultimately, I help make video games for a living. Thus, I would really like to use my custom compiler on some, well... games. This lead me down the long and complicated road of plugging my custom compiler into Unreal Engine. Turns out, it's not as easy as you'd think. When compiling a game for Windows, you have to use the MSVC compiler by default. There is no way around this using the publicly available version of Unreal. There is no option for clang. Thus, I was forced to rather cross-compile a Linux binary. I also had to alter some of the clang source code in order for Unreal Engine to play nicely. For example, I had to override the version string to trick Unreal into thinking my compiler version was clang 6.00 based. Apparently Unreal only supports certain versions of clang (or so they want you to think).

I thought about potentially cloning Clang and hosting another fork on my GitHub account with these Unreal-related changes, but I decided to rather just create a folder called `unreal-engine-clang-changes/` and add it to my LLVM fork. In this [folder](https://github.com/justiceadamsUNI/llvm/tree/ROP-Noop-Insertion-V2/unreal-engine-clang-changes) you can find the source files with the changes needed to make Unreal compile correctly using a custom built clang compiler. I'm hoping to eventually write up exactly how to do this, but I don't think that information belongs here (being that it's cumbersome). After a few tricks and some clang adjustments, I was finally able to compile some Unreal Engine samples using my compiler. Here's an example of the version.h file I needed to alter. Unreal will think our compiler is clang version 6.00 (it's not).
```
#define CLANG_VERSION 6.0.0
#define CLANG_VERSION_STRING "6.0.0"
#define CLANG_VERSION_MAJOR 6
#define CLANG_VERSION_MINOR 0
#define CLANG_VERSION_PATCHLEVEL 0
```

The exploration into Unreal, however, uncovered a greater issue. Unreal by default optimizes code for speed using the `-02` flag (as expected). This becomes a problem for no-op insertion because obviously, optimizers remove the noops. I discovered this after many hours of debugging using the GNU Binutils (mainly objdump). It's one of the simplest optimizations you could make when you really think about it. This creates a sort of trade-off between security and speed. I will document this in the future work section, although I'm not entirely sure I'll get around to tackling this problem in the scope of this project, being that it's rather complicated. We run into a similar problem at work: generating debug information with optimized code. Nonetheless, it is a problem to think about in the future. For now, I've disabled all optimizations in the compiler. This is not ideal but allows us to determine the effectiveness of the approach before tackling the "optimization" problem.

What I want to do now is analyze object code from Unreal samples using some of the more popular gadget finders. I would really like to test size increases in generated code and the number of gadgets that the tools find. Hopefully, the tools will find fewer gadgets when using the custom clang we've been working on. That brings us to the future work I'm planning on doing.


## For Next Week(s)...and maybe the far-flung future
- Adjust the noop insertion to use "Logical noops" rather than just the simple nop instruction. In this way, we might be able to better hide gadgets.
- Research Gadget finding tools and use them on our Unreal Engine object code which will be compiled using our custom compiler. This will allow us to analyze the effectiveness of the approach.
- Optimizing the code causes issues. This should be investigated more, although it is an incredible amount of work. We would probably need to create our own optimization level (and flag) which does all regular optimizations but keeps noop instructions.
- Clean up this repo with a better naming scheme for files. The dates are not easy to sift through anymore now that it's a new year. I done goofed.


## Sources
- [Inserting nodes into SelectionDAG (X86)](https://groups.google.com/forum/#!topic/llvm-dev/6L2Wfeh5K_A)
- [Profile-guided Automated Software Diversity](https://www.ics.uci.edu/~ahomescu/multicompiler_cgo13.pdf)
- [Compiling For Linux - Unreal Engine Wiki](https://wiki.unrealengine.com/Compiling_For_Linux#Getting_the_toolchain)
- [Building Linux cross-toolchain](https://wiki.unrealengine.com/index.php?title=Building_Linux_cross-toolchain)
- [Online disassembler](https://onlinedisassembler.com/odaweb/)
- [Clang Command Line Reference](https://clang.llvm.org/docs/ClangCommandLineReference.html)
- [Cross Compiling for Linux - Unreal Engine Docs](https://docs.unrealengine.com/en-us/Platforms/Linux/GettingStarted)
