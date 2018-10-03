# Week Of 9/24 Research Update
This week I spent my time watching an internal sony video about code security which highlighted ROP attacks and begun playing with some small exploitable applications so I could see how things worked firsthand. Here I will recap what I learned and what I hope to do next week. Unfortunately, I can't share any of the video's content's because it's from an internal SIE talk, but I will discuss the questions it prompted me to answer.

## Notes
Around half of my time, this week was spent simply watching a recording of an internal SIE discussion about code security. This presentation was given by PlayStation engineer Steve Osman and highlighted some of the things attackers do when exploiting code. I think Steve said it best when he said: "Hopefully, you can better appreciate the risk" with regards to understanding code exploits. In one sentence, he captured the spirit of this entire project. Without discussing the contents of the video, I'll try to cover some of the questions that arose in my head.

The stack grows downward as we all know, and to overwrite a return address (as you generally do in buffer overflow/ROP attacks) you generally need to use a buffer which grows upward, opposite the stack.

<p align="center">
  <img src ="https://www.usenix.org/legacy/publications/library/proceedings/sec98/full_papers/cowan/cowan_html/img1.gif" />
</p>

So why have buffers grow upward at all? If we had buffers grow downward, wouldn't that eliminate the vulnerability by ensuring no previous return addresses get overridden? Well...No.

<p align="center">
  <img src ="https://i.imgur.com/KGDWrun.jpg" />
</p>

After some initial research, I found the following [stack overflow post](https://security.stackexchange.com/questions/135786/if-the-stack-grows-downwards-how-can-a-buffer-overflow-overwrite-content-above) to be incredibly insightful. First, remember that the heap grows toward the stack. This means hackers would have the ability to override data on the heap, which opens up a whole new world of exploits.

Second, switching the direction of the buffers doesn't really accomplish anything. Consider the following piece of code that I took from the above stack overflow post
```
void foo(char *s)
{
    char buf[8];
    strcpy(buf, s);
    return;
}
```
What happens is that a buffer is created and then PASSED TO ANOTHER FUNCTION to be filled. This is incredibly common in the world of C++. To tie things into the world of game development, consider the following function signature from the [Unreal Engine API](http://api.unrealengine.com/INT/API/Runtime/Engine/FRenderTarget/ReadPixels/index.html)

```
bool ReadPixels(TArray < FColor > & OutImageData, FReadSurfaceDataFlags InFlags, FIntRect InRect)
```
This function is pretty useful for developers hoping to minimulate the players viewport with ease, and it also takes in a preallocated color buffer.

Going back to the foo example, what this means is that when `strcpy()` is called, the return address back to `foo()` is placed onto the stack, but it will soon be overwritten because a buffer overflow is imminent. Thus, switching directions is pretty useless to be honest... better luck next time for me. To better visualize this, consider the image from stack overflow
```
    stack grows to the right -->
    memory addresses increase to the right -->
            0x8000                          0x8010
------------++---------+----------+---------++-----------+-------------+
  ....      || char *s | ret addr | buf[8]  || ret addr  | locals  ... |
------------++---------+----------+---------++-----------+-------------+
 caller --->  <-------- foo() ------------->  <---- strcpy() ---------->
```

The second question that arose from this chat comes from the following: Consider the following ROP gadget as I discussed in my last [update](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-9-17-update.md)
```
pop rdi; 
ret
```
If you were trying to construct a ROP gadget chain, and if this is called from a function `foo()` (after highjacking the return pointer from whoever called foo), then isn't the value of `rdi` always going to be the pointer to foo() being that the return address is the last thing on the stack before we jump to this gadget?

In other words, we write a return address to this gadget meaning that we now jump from position A to position B in memory, but because of that jump instruction, isn't there now a return pointer to position A on the stack? Meaning that the `$rdi` register just points back to position A? If so, gadget chains would be basically impossible to construct.

Something about my current understanding of the way x86-64 handles return and calling instructions is wrong. I need to dive deeper into this next week.

As you can see, most of my week was spent watching an informational talk and researching the questions that arose out of that. As you probably saw earlier, I did start experimenting with executables from https://ropemporium.com/index.html, but I decided to shelve them while I get a better understanding of just what a ROP gadget chain is.

## For The Following Week(s)
Here are the questions raised/topics I want to explore next week:
- I need to answer my own question about the nature of ROP gadget chains. And how they can be used together. This is my primary goal for next week.
- Read the following public paper which was given to me by PlayStation employee Josh Magee. Apparently, it houses some useful information regarding ROP attacks which sparked an internal discussion at PlayStation.
- Reach out to Steve Osman and ask for any resources he has encountered in his exploration of buffer overflow attacks. This may prove useful.
- Continue to tinker with the ROPemporium examples to get a better understanding of how the exploit(s) works.

## Sources for this week:
Information sources
==========================
- [x64 Cheat Sheet - Brown University](https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf)
- SIE internal talk given by Steve Osman
- Stack Overflow (linked above)

Image Sources
==========================
- https://www.usenix.org/legacy/publications/library/proceedings/sec98/full_papers/cowan/cowan_html/img1.gif
- https://security.stackexchange.com/questions/135786/if-the-stack-grows-downwards-how-can-a-buffer-overflow-overwrite-content-above
