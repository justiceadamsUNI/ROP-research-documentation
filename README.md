# ROP-research-documentation
Repository housing all documentation/research for preventing ROP attacks in LLVM based compilers. All information will also live at www.justiceadams.com

<p align="center">
  <img src ="https://llvm.org/img/DragonMedium.png" />
</p>

# Goal
Throughout the 2018-2019 school year, I will be working with [Dr. Wallingford](wallingf@cs.uni.edu) on my senior research project which involves researching and preventing [ROP](https://en.wikipedia.org/wiki/Return-oriented_programming) (Return-oriented programming) exploits in LLVM/clang generated code. There exist more LLVM based compilers than clang, but I will be focusing on clang as it is a widely used C++ compiler. I've been interested in compilers ever since I took the compilers course at the University of Northern Iowa and built my own [compiler](https://github.com/justiceadamsUNI/Klein-Compiler) from scratch. Since then, I joined the toolchain team at PlayStation and have been working on toolchain systems for the PlayStation consoles used by game developers worldwide. At Sony, we develop and ship an internal LLVM based compiler to our developers. After speaking with some of the engineering managers, it became clear that these ROP exploits haven't been researched much, so I chose to dive in. Over the course of the year, I'm hoping to devise/improve methodologies which can reduce ROP attacks and hopefully help secure the PlayStation ecosystem. It's worth noting that although I'm receiving advise from engineers at Sony, **this project is not associated or funded by PlayStation in any way**. Furthermore, none of the code you'll see in this repo (or in some other clang mirror which I may host) will house Sony internal or confidential information. This is purely university research based on an open-source project which may be beneficial to PlayStation (and the hundreds of other organizations/projects using clang/LLVM).

<p align="center">
  <img src ="https://img.buzzfeed.com/buzzfeed-static/static/2015-08/8/11/enhanced/webdr04/anigif_enhanced-31027-1439049075-2.gif" />
</p>

# The Plan
Below is my plan of attack for this project. Note that any and all documentation will live here in this repo as well as on www.justiceadams.com. Hopefully, I can provide an update every week with relevant information and progress updates. As I work with Dr. Wallingford this plan is subject to change, but any changes will be documented appropriately.
- Research ROP attacks and tactics. I'm going to find out what causes the exploits and what attackers are looking for in generated target code. Hopefully, I can get a sense of the patterns they look for which will help later on.
- Research LLVM back-end code generation with an emphasis on how clang(the compiler driver) interacts with these pieces to generate an executable. Pairing this with my understanding of ROP attacks, I'm hoping to spot patterns in generated code.
- Devise/Improve tactics to reduce ROP attacks by altering LLVM source code.
- Test new/altered implementations with open source game projects to see if this is beneficial in the world of game development, which as I mentioned earlier, is my current field of work.

Aside from the rough outline I've laid out, I plan on reaching out to engineers at PlayStation, in the LLVM community, and at the [LLVM developers conference](https://llvm.org/devmtg/2018-10/) in October. Hopefully, I can meet with some others interested in my research!

Usefull links for the curious:
- [LLVM](https://llvm.org/)
- [Clang](http://clang.llvm.org/)
- [Clang Mirror](https://github.com/llvm-mirror/clang)


# Results
After a year of research, I've successfully managed to reduce ROP gadget detection in LLMV-compiled games. I wrote a pretty extensive blog post over on my website where I discuss the process, issues, and results. For a more concise representation of the project, I encourage you to go check it out [here](http://www.justiceadams.com/blog/2019/5/28/can-we-prevent-rop-attacks-in-llvm-compiled-games)

If you want a more verbose reading, feel free to read through the documentation in this repo
