# Week Of 10/1 Research Update
Last week I was left with a question regarding ROP gadget chains, and just exactly how they worked. This lead me down a path of research regarding x86 architecture and how return statements are executed. I was able to find the answer to my question and further understand the nature of ROP gadget chains in the process. After that, I played around with an exploitable executable file and was able to successfully complete my first buffer overflow attack.

## Notes
Last week I was stumped with regard to how you can string together chains of ROP gadgets. It made sense when you explore how to access one chain, but not when you wanted to string together multiple statements. To better understand this I first had to explore x86-64 architecture. After hours of confusion, I finally decided to get out the ol' pen and paper and draw out a function stack. After doing this, I had a bit of a revelation (along with some research to help).

<p align="center">
  <img src ="https://i.imgur.com/8EDzP1A.jpg" />
</p>

I'll try to make clear what my chicken scratch represents above. First, let's look at how return statements work in x86. `rsp` is the register which points to the top of the stack (it's the stack pointer). When a return statement is executed it adjusts the stack pointer accordingly. 

```
"ret loads the value rsp points to into rip and increases rsp by 8 bytes, mimicking a pop rip instruction." - Andreas Follner
```

Essentially, a return instruction mimics a `pop rdi;` instruction: popping the top of the stack into the `rdi` register which is your instruction pointer (AKA where the program will jump to to find instructions to execute). It increases the `rsp` (stack pointer) by 8 bytes because we're talking about a 64-bit architecture, where each memory address is 8 bytes.

What this means is that if you wanted to string together a chain of rop gadgets, you simply have to manipulate the stack in a certain way such that each of your gadgets is called in order. A hacker would take the following steps.

1.) Locate all return instructions in a programs assembly. This can be done using binutils/special programs.
2.) Walk backward from every `ret` instruction to find particular instructions of interest that live right next to the `ret` instruction. These are your ROP gadgets.
3.) Document every ROP gadget you found and devise some way to string them together to do what you want. Ex: "If I execute gadget 1, gadget 255, and gadget 291 in that order I'll have accessed the systems' shell"

**Note:** For steps two and three, there are tools that do this for you which I'm hoping to explore later. See my plan for the following week(s). Noone is really searching through the assembly by hand (unless you're researching a small executable as I did).
4.) Manipulate the stack in a way such that your gadgets get executed.

<p align="center">
  <img src ="https://i.imgur.com/ft8SxW6.png" />
</p>

Excuse my paint skills for a second and consider the following stack. Let's say our program had a buffer which was basically unprotected. We could write up to the return address value and then past that for the next two memory values and we could put all three addresses to the gadget instructions on the stack. Remember that each gadget ends in a return instruction: so gadget 1 gets executed ending in a `ret` instruction. This causes gadget 255 to be executed which also ends in a `ret` instruction. This causes gadget 291 to be executed thus giving us the shell we wanted (in this hypothetical). The main thing to remember is that the key steps are **a.)** finding the addresses of all the gadgets which **ALREADY EXIST IN MEMORY** since they are part of the original program, and **b.)** manipulating your program to execute those gadgets when needed. 

If you need a bigger buffer to exploit, you could potentially jump to a standard library function if you found its address. You could even let the program run until it gets to a bigger buffer you can exploit if that helps. Ex:
```
1.) rewrite a return address so a gadget is executed
2.) let the original program run for a bit so we can get to a different buffer exploit
3.) jump to another gadget
repeat
```

It's worth noting that you can place arguments on the stack as well which will be popped into registers before a function call: meaning we can use certain gadgets to place arguments in registers followed by a jump to certain function instructions, effectively simulating a function call. It becomes clear pretty quickly that the possibilities are somewhat endless.

<p align="center">
  <img src ="https://i.imgur.com/uJQo50u.png" />
</p>

These gadgets are executed once they are already in memory, meaning that we manipulate the stack to execute pieces of code that are already loaded into memory for the most part*. In an ideal world, you use the exploitable application to do your bidding. This is partly because most operating systems have stack execution protection mechanisms. There's a certain spot in memory that can run executable code, and if you're familiar with the way a C programs memory works you'll know that it's in a different place than the stack section we manipulate.

<p align="center">
  <img src ="https://cdncontribute.geeksforgeeks.org/wp-content/uploads/memoryLayoutC.jpg" />
</p>


## Exploiting a buffer overflow
I also spent some time this week exploiting my first buffer overflow with an example from ROP Emporium. Unfortunately, the executable was compiled for Linux so I had to do some workarounds to get it to work on windows.

Using Linux [binutils](https://www.gnu.org/software/binutils/) and an application called [Radare2](https://rada.re/r/) I was able to see that the application allocates 32 bytes of memory to a standard buffer
![allocation](https://i.imgur.com/U5HaTyO.png)
The goal is to use this programmer-mishap to overwrite the return address on the stack in a way that lets us jump to a function that isn't intended to be called!

We know the buffer is 32 bytes (and it's the only local variable), but because this function pwnme() which you see above was called from main, there is another value on the stack which is the saved base pointer. The processor uses this by adding and subtracting to find necessary variables. And because this is a 64-bit architecture, that's another 8 bytes on the stack in our way. So we have a 32-byte buffer, 8-byte base pointer, then the return address which we wish to overwrite.

<p align="center">
  <img src ="https://i.imgur.com/0Axn628.png" />
</p>
Ignore the fact that the stack is growing the wrong way in the above picture...

So what this means is that we need 40 bytes of garbage, and then our return address which I found out was the address `0x0040081f` using radare2. Bear with me. So we need to put this address in the register in bytes. There are two digits of hex per one byte, so in python, this address is `\x00\x40\x08\x1f` but we need 8 bytes since it's an 8-byte address, so we pad the value to be `\x00\x40\x08\x1f\x00\x00\x00\x00`. I want this value to be piped into the return address slot of the stack. **BUT** this is not entirely correct. After **hours** of debugging segfaults (which is what happens when you jump to memory you aren't supposed to be able to) apparently, there is something called [endianness](https://en.wikipedia.org/wiki/Endianness) which affects the way bytes are stored in memory. In short... bytes are reversed by intel's industry standard. Sigh.... Thus, I simply needed to pipe `\x1f\x08\x40\x00\x00\x00\x00\x00` to my executable to jump to the correct function!

<p align="center">
  <img src ="https://i.imgur.com/gZfjnaU.png" />
</p>

I figured this out by using a python library called [pwntools](https://github.com/Gallopsled/pwntools) which surprised me when I told it to use the given memory address, and it reversed all the bytes. This lead me on the path to discovering endianess. Fun stuff.


## For Next Week
- Every time I run this exploitable executable, the address of the function in memory is the exact same: `0x0040081f`. Is this normal? Shouldn't it be random or something to prevent what I'm doing? I need to investigate this further
- Reach out to Steve Osman to ask for more resources
- Continue with the ROP emporium exercises before moving onto LLVM architecture. Particularly, get to the point where I actually have to find chains and string them together, as you would in a real attack
- Stay sane while trying to debug binaries.

## Sources for this week
### Information sources:
- [PwnTools](https://github.com/Gallopsled/pwntools)
- [ON GENERATING GADGET CHAINS FORRETURN-ORIENTED PROGRAMMING](https://bodden.de/pubs/phd-follner.pdf)
- [Return Oriented Programming (ROP) Exploits Explained](https://www.rapid7.com/resources/rop-exploit-explained/)
- [x86 Call/Return Protocol - University of Wisconsin](http://pages.cs.wisc.edu/~remzi/Classes/354/Fall2012/Handouts/Handout-CallReturn.pdf)
- [x64 Cheat Sheet - Brown University](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf)
- [Return-Oriented Programming (Jonathan Zentgraf) - Cybersecurity Club Presents](https://www.youtube.com/watch?v=CbW5TYmWQNU)
- [Radare2 cheatsheet](https://gist.github.com/williballenthin/6857590dab3e2a6559d7)
- [Ret2Win challenge](https://ropemporium.com/challenge/ret2win.html)

### Image Sources
- https://camo.githubusercontent.com/16a4295c576d14f9e2ed05ca88976697d068ff65/68747470733a2f2f63646e636f6e747269627574652e6765656b73666f726765656b732e6f72672f77702d636f6e74656e742f75706c6f6164732f6d656d6f72794c61796f7574432e6a7067
- https://www.youtube.com/watch?v=CbW5TYmWQNU
