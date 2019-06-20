# Week Of 9/17 Research Update
This was the first week of my ROP related research. Things went fairly smoothly, but I had to postpone my write-up due to the fact that I was traveling all last weekend. My main objective last week was to explore just what ROP (return oriented programming) is and how it works. I'll try to recap what I learned and what I'm hoping to do in the coming week with respect to the project.

**Note:** I will assume you have a decent understanding on general data structures.

## Notes
So return-oriented programming is a buffer based attack which uses the ability to overwrite certain memory addresses to take control of peoples machines. Particularly, you aim to overwrite return addresses in generated code. If an attacker can overwrite the value of a return address on the runtime stack, they have the opportunity to point that address to their own code, and in turn, execute their own malicious program sequence. If you think about a computer stack it traditionally grows "down", meaning that if you put something onto the stack, then you have lowered the address of the stack pointer.

<p align="center">
  <img src ="https://eli.thegreenplace.net/images/2011/02/stack1.png" />
</p>

Recall also, that whenever a function is called a return address is placed onto the stack. This return address tells the program where to return to when it's time to pop a stack-frame off of the stack. Consider the case in the following image, where a buffer exists below the return address. This buffer can then be overwritten (being that buffers grow upward traditionally) to overwrite the return address. If we somehow had shellcode on the stack in a different address, we could jump to it and create an exploit. Shellcode is a small bit of assembly code, usually put in memory that can be executed directly.

<p align="center">
  <img src ="https://www.usenix.org/legacy/publications/library/proceedings/sec98/full_papers/cowan/cowan_html/img1.gif" />
</p>

This exploit allows you to execute your own code, and potentially even call some other code like standard library functions. You can see why this would give the attacker a ton of power. Attackers generally look for things called ROP gadgets. These gadgets are small pieces of backend code next to return statements. They provide a small piece of code which can perform a certain task which can be useful. For example, the following is considered a gadget in x86 architecture.
```
pop rdi; ret
```
This small piece of code pops the top of the stack into the rdi register and returns to whatever the return address is. If someone was able to override the return address value, they would basically be able to use the rdi register as a way to store an argument before calling any code they wanted. I need to do more research next week regarding how x86 architectures handle their return and call sequences. I have some experience implementing this functionality in my Klein compiler, so I have a good idea of how it works, but I need to further explore how this works in an actual widely used machine language (x86). For now, I've been referring to the following documentation for help:
- [x86 Call/Return Protocol - University of Wisconsin](http://pages.cs.wisc.edu/~remzi/Classes/354/Fall2012/Handouts/Handout-CallReturn.pdf)
- [x64 Cheat Sheet - Brown University](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf)

Being able to string these "gadgets" as they are called together opens up the applications host system for manipulation. These gadget chains are what attackers look for when trying to exploit code. In the future, I'm hoping to come up with some algorithm which makes detecting ROP gadgets much harder, thus making gadget chains much harder to construct.

There are some ways around this exploit apparently. Some systems make the stack non-executable with NX protection while some randomize all the memory addresses on the stack to make things much harder (AKA local variables aren't right next to the return address and so forth) via Adress Space Layout Randomization. Admittedly, I need to look more into this, especially since my goal is to secure LLVM compilers.

<p align="center">
  <img src ="http://4.bp.blogspot.com/-9lCjBfSF9Kk/TsHAMicF3VI/AAAAAAAAACg/VN5K2XFz57w/s1600/ROP-stack-spray-diagram.png" />
</p>


## For The Following Week(s)
Here are the questions raised/topics I want to explore next week:
- I want to continue to explore the nature of ROP attacks. At this point, I really want to find time to go hands-on and work with a small exploitable application to see how things work. I found [ROP emporium](https://ropemporium.com/index.html) online and am hoping to work through an example or two.
- After some discussions with senior engineers at PlayStation (Josh Magee and Brendan Rehon), I'm planning to watch an internal recorded security technical discussion given by Steve Osman. Hopefully, I can find some useful information about what ROP attacks are, and what he thinks are good ways to prevent them. I'm also planning on reaching out to him for advice, so stay tuned.
- Read the following public paper which was given to me by PlayStation employee Josh Magee. Apparently, it houses some useful information regarding ROP attacks which sparked an internal discussion at PlayStation.
- Continue to explore x86 architecture and understand just what makes a ROP gadget. I'm still a bit shaky on what exactly it means to "chain" these gadgets together. Most if not all of my work will be in x86 architectures, so it would be good to brush up.
- Explore tools to help find these ROP gadgets. There's no way people are reading assembly code on their own, so I'd like to use and learn the most popular tools for discovering the gadgets hackers look for.

## Sources for this week:
Information sources
==========================
- [Introduction to return oriented programming (ROP) - YouTube](https://www.youtube.com/watch?v=yS9pGmY_xuo)
- [Return Oriented Exploitation (ROP) - YouTube](https://www.youtube.com/watch?v=5FJxC59hMRY)
- [Return-Oriented Programming (Jonathan Zentgraf) - Cybersecurity Club Presents - YouTube](https://www.youtube.com/watch?v=CbW5TYmWQNU)
- [64-bit Linux Return-Oriented Programming - Stanford](https://crypto.stanford.edu/~blynn/rop/)
- [x86 Call/Return Protocol - University of Wisconsin](http://pages.cs.wisc.edu/~remzi/Classes/354/Fall2012/Handouts/Handout-CallReturn.pdf)
- [x64 Cheat Sheet - Brown University](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf)
- [64-bit Linux stack smashing tutorial: Part 1](https://blog.techorganic.com/2015/04/10/64-bit-linux-stack-smashing-tutorial-part-1/)
- Conversations with Senior Enineers at PlayStation (Brendan Rehon and Josh Magee)

Image Sources
==========================
- http://4.bp.blogspot.com/-9lCjBfSF9Kk/TsHAMicF3VI/AAAAAAAAACg/VN5K2XFz57w/s1600/ROP-stack-spray-diagram.png
- https://www.usenix.org/legacy/publications/library/proceedings/sec98/full_papers/cowan/cowan_html/img1.gif
- https://eli.thegreenplace.net/images/2011/02/stack1.png
