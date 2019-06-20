# Week Of 1/7 New Years Research Update
The Spring semester has started back up, and I've gone back to my research in full force. In my [last update](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-11-5-update.md), I mentioned how I had been learning about LLVM backend development, including things like LLVM-IR, SelectionDags, and instruction lowering. This week, I first did some reading about a potential approach to making gadgets hard to find, and then spent some time looking at return instructions in the interwebs of the LLVM backend.


## Introducing No-Ops
I spent a good amount of time over the last week reading the following [paper](https://www.ics.uci.edu/~ahomescu/multicompiler_cgo13.pdf) titled "Profile-guided(https://www.ics.uci.edu/~ahomescu/multicompiler_cgo13.pdf) Automated Software Diversity". In short, the paper covered a brief introduction to memory based (ROP) attacks and introduced a potential solution. The paper introduced the idea of inserting Noops between each instruction to make gadgets hard to find, while still producing equivalent binaries.

```" Borrowing a term from biology, we refer to the prevalent practice of shipping identical binaries to all customers as the software monoculture. "```

I think that's important, we don't want to alter the resulting executable/binary for end-users: we just want to make the product safer. In this way, the authors of the paper went on to randomly insert no-ops inside their target code. By inserting these instructions, the goal would be to make ROP gadgets harder to find. "The ultimate defense is to drive the complexity of the attack". 

Interestingly enough, they came to the same conclusion that I did: The tablegen portion of the LLVM backend is a horrible place to insert instructions and is incredibly non-user-friendly. I really like this diagram they use. Not only does it help visualize the steps associated with compilation in the LLVM compiler, but it's a good representation of where to insert noops.

<p align="center">

  <img src ="https://i.imgur.com/PpYKbhm.png" />

</p>


There's a lot to like about this paper. It highlights the reasons why you can't insert the instructions at a higher level (because they might be removed via optimization) and nicely lays out their approach. I, however, would like to tweak their method. I think we can prevent gadget detection by inserting no-ops **only** around the return instructions. By making the instructions next to a return instruction harder to see, we can make gadgets harder to find. This will be the approach going forward

```" Therefore, our strategy is to insert NOPs into the lower-level representation, after the compiler performs all optimizations and just before it emits native code."```


## Altering The Return Instruction
In order to alter the return instruction, I first had to understand how it was being handled. It turns out, that there is an entire function for handling the return instruction when SelectionDag's are lowered to their respective infrastructure. The method is appropriately called `X86TargetLowering::LowerReturn`. In here, I need to find a way to alter the Instruction chain to successfully include a random number of No-ops without changing the logic of the resulting binary. You pass in a chain, it gets altered accordingly as to lower the return instruction for x86 architectures, and then it gets returned. 

I need to change the line 

```RetOps[0] = Chain;```

within X86ISelLowering.cpp to a different chain. I need to figure out the way to alter the chain in an elegant manner. I'm very close, but there are q few glitches. I haven't figured this out quite yet so that is my goal for next week. From there, I'll put my OOP cap on and devise a clean way to handle this (presumably with an object or two). For more on that function mentioned above, refer to the [documentation](http://llvm.org/doxygen/X86ISelLowering_8cpp_source.html).


## For next week
- Figure out how to alter the chain of instructions in the `LowerReturn()` function
- Devise/plan a higher level OOP approach for inserting no-ops for gadget prevention
- Re-enroll in the research course


## Sources
- [Inserting nodes into SelectionDAG (X86)](https://groups.google.com/forum/#!topic/llvm-dev/6L2Wfeh5K_A)
- [SelectionDAG](http://llvm.org/doxygen/classllvm_1_1SelectionDAG.html)
- [Gluing arbitrary nodes together](http://lists.llvm.org/pipermail/llvm-dev/2016-June/100885.html)
- [Profile-guided Automated Software Diversity](https://www.ics.uci.edu/~ahomescu/multicompiler_cgo13.pdf)
