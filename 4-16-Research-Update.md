# Week Of 3/24 Research Update
Last time, I left off with a looming question "Why ar new gadgets showing up in my program with my V4 compiler?". I examined the gadgets over and over until eventually I came to a conclusion about x86 architecture, it must be the case that you can "splice" these instructions. This led me to abandon the logical nop approach until I recieved further validation (I needed to talk to an x86 expert). Well, as chance would have it, I got to speak to many.

# Euro LLVM

<p align="center">
  <img src ="https://cdn-az.allevents.in/banners/19f201c0-019b-11e9-93f2-875866379563-rimg-w1200-h600-gmir.jpg" />
</p>

Last week, I had the opportunity to travel to Europe for the EuroLLVM conference with the PlayStation toolchain team. While there, I managed to speak to a few developers about my issue in hopes for a solution. I managed to get ahold of Paul Robinson, one of PlayStations senior compiler engineers. When I told him about my issue when adding logical nops, he said something that will probably stay with me forever.

```"The worst day of my entire life was when I got hired at PlayStation, why? Because the recruiter said "it's X86""```

We then went on to talk about how logical nops will never work (at least not as well as we'd hope), because the x86 architecture manages to embed instructions inside other instructions. What this means is that, say for example, you have an insruction that takes up 3 bytes. For this example, let's assume it's an add instruction. Well, if you start executing the instruction from the second byte, you may actually be executing a completely different instruction! (maybe a multiply, maybe a register load, etc). The simple fact is, that you can "splice" these instructions to other instructions. Thereforce, if we insert a logical nop before a return instruction, it will actually introduce more gadgets! That's pretty gross. Here's a visual which may help you understand

<p align="center">
  <img src ="https://i.imgur.com/eVLfdV2g.jpg" />
</p>

In the visual above, the green bracket represents the instruction we would insert (in our case a logical nop), but the black bracket represents the new instruction we've introduced if you start executing at byte 2. This is obviously a problem, as we've just introduced a new gadget. This is the primary reason I've abandoned the logical nops (to an extent)....


# V5
I've pushed a V5 of the compiler to github [here](https://github.com/justiceadamsUNI/llvm/tree/ROP-Noop-Insertion-V5). Among other things, this version goes back to using base nop instructions for the reason outlined above, handles call instrucitons by padding those as well (as they may introduce gadgets), tweaks the level of randomness for nop insertion (between 10-15 nops for each insertion), and cleans the code a little bit. This is going to be the primary version of the compiler I will be riding with for the rest of the year, but it's not the only one...


# V4.5
Yes, I've also have a v4.5 verison of the compiler built locally. This adds nops into the stream just as v5 does, but it also adds 2-3 logical nops at random stages in the stream. The idea is that, this will allow us to properly "hide" the last user instruction within the stream. If using v5, the instruction before the nops is obviously the last valid instruction. But if we introduce some logical nops we end up with a stream somewhat like this
```
-LOGICAL NOP-
-ACTUAL NOP-
-ACTUAL NOP-
-VALID INSTRUCTION-
-ACTUAL NOP-
-ACTUAL NOP-
-LOGICAL NOP-
-RETURN-
```
This type of stream makes it hard for an attacker (and the tool he uses) to find which instruction actually is the users valid instrucion when determining a gadget. The problem however, as we outlined earlier, is that this might (and probably will) actually introduce more gadgets into the stream. The idea is that it might introduce less than it prevents. Well see when I gather more throughout measurements. Hopefully, I'll have v4.5 pushed to github soon!

# Initial results for the final compilers
After tweaking the compiler and measuring results accurately, I've managed to get some numbers! These are initial numbers, and more need to be collected before the end of the semester, but it's a start!

### Klein Compiler
When compiling the klein compiler I see the following results
- V5.0 of the compiler decreses gadget counts by 28% with only a 5% increase in object code size
- V4.5 of the compiler decreases gadget counts by 16% with a 8% increase in object code size.

### Unreal Engine Infiltrator Demo
- V5.0 of the compiler decreses gadget counts by 7% with only a 1% increase in object code size# Week Of 3/24 Research Update
Last time, I left off with a looming question "Why are new gadgets showing up in my program with my V4 compiler?". I examined the gadgets over and over until eventually, I came to a conclusion about x86 architecture, it must be the case that you can "splice" these instructions. This led me to abandon the logical nop approach until I received further validation (I needed to talk to an x86 expert). Well, as chance would have it, I got to speak to many.

# Euro LLVM

<p align="center">
  <img src ="https://cdn-az.allevents.in/banners/19f201c0-019b-11e9-93f2-875866379563-rimg-w1200-h600-gmir.jpg" />
</p>

Last week, I had the opportunity to travel to Europe for the EuroLLVM conference with the PlayStation toolchain team. While there, I managed to speak to a few developers about my issue in hopes of a solution. I didn't get a solution, but I got valuable advice. I managed to get ahold of Paul Robinson, one of PlayStation's senior compiler engineers. When I told him about my issue when adding logical nops, he said something that will probably stay with me forever.

```"The worst day of my entire life was when I got hired at PlayStation. Why? Because the recruiter said "it's X86""```

We then went on to talk about how logical nops will never work (at least not as well as we'd hope) because the x86 architecture manages to embed instructions inside other instructions. This quickly devolved into us bashing x86 as intoxicated software engineers do. What this means is that, say, for example, you have an instruction that takes up 3 bytes. For this example, let's assume it's an add instruction. Well, if you start executing the instruction from the second byte, you may actually be executing a completely different instruction! (maybe a multiple, maybe a register load, etc). The simple fact is, that you can "splice" these instructions to other instructions. Therefore, if we insert a logical nop before a return instruction, it will actually introduce more gadgets! That's pretty gross. Here's a visual which may help you understand

<p align="center">
  <img src ="https://i.imgur.com/eVLfdV2g.jpg" />
</p>

In the visual above, the green bracket represents the instruction we would insert (in our case a logical nop), but the black bracket represents the new instruction we've introduced if you start executing at byte 2. This is obviously a problem, as we've just introduced a new gadget. This is the primary reason I've abandoned the logical nops (to an extent)....


# V5
I've pushed a V5 of the compiler to GitHub [here](https://github.com/justiceadamsUNI/llvm/tree/ROP-Noop-Insertion-V5). Among other things, this version goes back to using base nop instructions for the reason outlined above, handles call instructions by padding those as well (as they may introduce gadgets), tweaks the level of randomness for nop insertion (between 10-15 nops for each insertion), and cleans the code a little bit. This is going to be the primary version of the compiler I will be riding with for the rest of the year, but it's not the only one...


# V4.5
Yes, I've also had a v4.5 version of the compiler built locally. This adds nops into the stream just as v5 does, but it also adds 2-3 logical nops at random stages in the stream. The idea is that this will allow us to properly "hide" the last user instruction within the stream. If using v5, the instruction before the nops is obviously the last valid instruction. But if we introduce some logical nops we end up with a stream somewhat like this
```
-LOGICAL NOP-
-ACTUAL NOP-
-ACTUAL NOP-
-VALID INSTRUCTION-
-ACTUAL NOP-
-ACTUAL NOP-
-LOGICAL NOP-
-RETURN-
```
This type of stream makes it hard for an attacker (and the tool he uses) to find which instruction actually is the users valid instruction when determining a gadget. The problem, however, as we outlined earlier, is that this might (and probably will) actually introduce more gadgets into the binary. The hope is that it might introduce fewer gadgets than it prevents, thus making it overall beneficial. Wel'l, see when I gather more throughout measurements. Hopefully, I'll have v4.5 pushed to GitHub soon!

# Initial results for the final compilers
After tweaking the compiler and measuring results accurately, I've managed to get some numbers! These are initial numbers, and more need to be collected before the end of the semester, but it's a start!

### Klein Compiler
When compiling the Klein compiler I see the following results
- V5.0 of the compiler decreases gadget counts by 28% with only a 5% increase in object code size
- V4.5 of the compiler decreases gadget counts by 16% with an 8% increase in object code size.

### Unreal Engine Infiltrator Demo
- V5.0 of the compiler decreses gadget counts by 7% with only a 1% increase in object code size
<p align="center">
  <img src ="https://docs.unrealengine.com/portals/0/images/Resources/SampleGames/StrategyGame/StragetyGame.png" />
</p>

As discussed in a [previous post](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-2-4-to-2-18-Research-Update.md), there are components contributing to the Unreal Pak (and object files) Files, so a 7% decrease in gadgets isn't horrible, it's actually not bad. It's not the 10% target we originally set out for, but it's close.


# For The Next Month
We're at the point that the project is wrapping up. So I'll try to outline what I would like to get done for the next month.
- Gather results. Compile more projects (including unreal projects) and record their results
- Ultimately, I don't feel like nop insertion is the best approach for preventing memory attacks. Many other developers seem to agree with me. I'm hoping to briefly research some other methods which I could present as alternatives, and I've already got some interesting leads from the various devs at EuroLLVM
- Clean/Upload the script I've been using to automate measuring many object files at once
- Document what it would take to move these changes upstream (even though I don't recocomend doing so)

Ultimately, This is the body of work I would like to get done before construction my final presentation which will take place on May 10th!

# Sources
- Paul Robinson of PlayStation
- Tom Weaver of PlayStation
- Brendan Rehon of PlayStation
- Josh Magee of PlayStation
- [Elf Binary Format Source File](http://llvm.org/doxygen/BinaryFormat_2ELF_8h_source.html)
- [Objdump Manual](https://sourceware.org/binutils/docs/binutils/objdump.html)
- [DrawIo](https://www.draw.io/)
- [ROPGadget](https://github.com/JonathanSalwan/ROPgadget)
<p align="center">
  <img src ="https://docs.unrealengine.com/portals/0/images/Resources/SampleGames/StrategyGame/StragetyGame.png" />
</p>

As discussed in a [previous post](https://github.com/justiceadamsUNI/ROP-research-documentation/blob/master/Week-of-2-4-to-2-18-Research-Update.md), there are numerous things that make up the Unreal Pak Files, so a 7% decrease in binary isn't horrible. It's not the 10% target, but it's close.


# For The Next Month
We're at the point that the project is wrapping up. So I'll try to outline what I would like to get done for the next month.
- Gather results. Compile more projects (including unreal projects) and record their results
- Ultimately, I don't feel like nop insertion is the best approach for preventing memory attacks. I'm hoping to research some other methods, and I've already got some leads from the various devs at EuroLLVM
- Clean/Upload the script I've been using to automate measuring many object files at once
- Document what it would take to move these changes upstream (even though I don't recocomend doing so)

Ultimately, This is the body of work I would like to get done before construction my final presentation which will take place on May 10th!

# Sources
- Paul Robinson of PlayStation
- Tom Weaver of PlayStation
- Brendan Rehon of PlayStation
- Josh Magee of PlayStation
- [Elf Binary Format Source File](http://llvm.org/doxygen/BinaryFormat_2ELF_8h_source.html)
- [Objdump Manual](https://sourceware.org/binutils/docs/binutils/objdump.html)
- [DrawIo](https://www.draw.io/)
- [ROPGadget](https://github.com/JonathanSalwan/ROPgadget)
