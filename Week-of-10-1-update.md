# Week Of 10/1 Research Update
Last week I was left with a question regaurding ROP gadget chains, and just exactly how they worked. This lead me down a path of research regaurding x86 architecture and how return statements are executed. I was able to find the answer to my question and further understand the nature of ROP gadget chains in the process. After that, I played around with an exploitable executable file and was able to successfully complete my first buffer overflow attack.

## Notes
Last week I was stumped with regaurds to how you can string together chains of ROP gadgets. It made sense when you explore how to access one chain, but not when you wanted to string together multiple statements. To better understand this I first had to explore x86-64 architecture. After hours of confusion I finally decided to get out the ol' pen and paper and draw out a function stack. After doing this, I had a bit of a revlation.

I'll try to make clear what my chicken scratch represents above. First let's look at how return statements work in x86. rsp is the register which points to the top of the stack (it's the stack pointer). When a return statment is executed, it adjust the stack pointer accordingly. 

```
"ret loads the value rsp points to into rip and increases rsp by 8 bytes, mimicking a pop rip instruction." - Andreas Follner
```

Essentially, a return instruction mimicks a `pop rdi;` instruction: popping the top of the stack into the `rdi` register. It increases the `rsp` (stack pointer) by 8 bytes because we're talking about a 64 bit architecture, where each memory adress is 8 bytes.

What this means is that if you wanted to string together a chain of rop gadgets, you simply have to manipulate the stack in a certian way such that each of your gadgets is called in order. A hacker would take the following steps.

1.) Locate all return instructions in a programs assembly. This can be done using binutils.
2.) Walk backward from every `ret` instruction to find particular instructions of interest that live right next to the `ret` instruction. These are your ROP gadgets.
3.) Document every ROP gadget you found and devise some what to string them together to do what you want. Ex: "If I execute gadget 1, gadgtet 255, and gadget 291 in order I'll have accessed the systems shell"

**Note:** For steps two and three, there are tools that do this for you which I'm hoping to explore later. See my plan for the following week(s). Noone is really searching through assembly by hand (unless you're researching a small executable like I did).
4.) Manipulate the stack in a way such that your gadet's get executed.

![ROP chain](https://i.imgur.com/ft8SxW6.png)

Excuse my paint skills for a second and consider the following stack. Let's say our program had a buffer which was basically unprotected. We could write up to the return adress value and then past that for the next two memory values and we could put all three adresses to the gadget instuctions on the stack. Rmember that each gadget ends in a return instruction: so gadget 1 gets executed ending in a `ret` instruction. This causes gadget 255 to be excecuted which also ends in a `ret` instruction. This causes gadget 291 to be executed thus giving us the shell we wanted (in this hypothetical). The main thing to remember is that the key steps are **a.)** finding the adress of all the gadgets which **ALREADY EXIST IN MEMORY** since they are part of the original program, and **b.)** manipulating your program to point to those gadgets when neeeded. 

If you need a bigger buffer to exploit, you could potentially jump to a standard library function if you found it's adress. You could evenn let the program run until it gets to a bigger buffer you can exploit if that helps. Ex:
```
1.) rewrite a return adress so a gadget is executed
2.) let the original program run for a bit so we can get to a different buffer exploit
3.) jump to another gadget
repeat
```

It's worth noting that you can place arguments on the stack as well which will be popped into registers before a function call, meaning we can use certain gadgets to place arguments in registers followed by a jump to certain function instructions, effictively simulating a function call. It becomes clear pretty quickly that the possibilities are somewhat endless.
![simulating a call to foo](https://i.imgur.com/uJQo50u.png)
