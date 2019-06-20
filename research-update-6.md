# Weeks Of 11/19 to 12/2 Research Update
Well, it's been a while. I basically took Thanksgiving week off, then jumped back into research. I've been staring at LLVM backend code generation source for the past couple of weeks with a very simple goal in mind. Turns out, the LLVM backend is complex...very complex. I've poured over the code for a considerable amount of time before just reaching my solution yesterday. Here lies the reason that this update has been delayed for so long. In this (probably long) update, I will explain the following

- My original goal and why I wanted to do it
- The problem that arose
- The solution
- Where I go from here

Let's dive right in.

# My original goal
The goal was simple. Alter a piece of backend code in a meaningful way to get a better understanding of how everything works together. I wanted to take a piece of assembly and essentially "swap out" an instruction for another. I thought this would be a good exercise in exploring the LLVM backend, being that the backend is primarily where I think my ROP prevention will take place from what I've learned so far. In the future, `ret` statements will become increasingly important, so I wanted to see just how everything was glued together. Out of my own ignorance, I guess I truly thought I was going to be able to replace a line of code such as 

```
if AST.Token == "ADD" {
  outputAssemblyInstruction("add")
}
```

with a line of code such as

```
if AST.Token == "ADD" {
  outputAssemblyInstruction("sub")
}
```

And poof, I would have replaced all add instructions with subtraction instructions. Maybe this comes from the simplistic way I wrote my [klein backend](https://github.com/justiceadamsUNI/Klein-Compiler/blob/master/Compiler/src/implementation/InstructionManager.cpp#L103), which had no IR. At this point, you can already tell that it wasn't that easy. Turns out, there's a lot going on in the LLVM backend, but the good news is that I left with a greater understanding of how clang works. Now, let's talk about the problem that occurred.


# The Dreaded Problem
I spent a good amount of time just reading about the LLVM backend. Reading the documentation on the LLVM website, people's blogs, and frankly just diving through the source code. Eventually, I developed the following understanding of how LLVM generates the code for my x86-64 machine (All of my sources are listed at the bottom of this page). This is somewhat simplified but gives a good idea of the sequence of events
- First, the front end (clang, not the clang driver) produces an AST tree that represents the given program in question
- At this point, the IR is converted to LLVM bytecode where it is used to create the initial SelectionDAG (The selectionDAG is a directed graph which represents another form of Intermediate representation). The SelectionDAG is an IR form which is "lower" than the LLVM IR. It contains information relevant to the target machine that is used when selecting registers, instructions etc. 
- The selectionDag is then "lowered" and legalized, meaning that every type in the DAG is legal for the specific architecture and any instructions which require explicit optimizations for a given architecture are taken into account.
- From there instructions are selected, any target-specific implementation C++ code is executed (to generate an appropriate instruction), and this DAG is used to eventually generate the assembly file which will go on to get linked into our final executable.

```"A legal DAG for a target is one that only uses supported operations and supported types - LLVM documentation"```

So my approach was straightforward, I'll alter the part that matches DAG nodes to assembly code in order to alter the ADD instruction. This approach led me to the discovery of [TableGen](https://llvm.org/docs/TableGen/index.html) which will haunt my dreams for years to come. Basically, the LLVM developers realized that much of the backend code is duplicated, so much so, that they built a framework where you can simply define instructions and their values, operators, and attributes in table definition files (`.td`) and then the TableGen tool generates the backend C++ code automatically for you. After a few hours tinkering with these tables, I abandoned this plan. All of the tablegen code is so intrinsically tied together, that if you alter one instruction chaos will ensue. Instead, I decided to take the approach of altering the DAG. The DAG is close enough to the machine representation of the machine, that it seemed like a good spot to manipulate. Every node gets piped through a `visit()` in the DAG builder so my approach was simple: find where the visitor handles an ADD instruction as part of the DAG, and alter the source to instead handle a SUB node. First I wrote a simple application in C++
```
int foo(int a, int b, int c) {
  int sum = a + b;
  return sum / c;
}

int main() 
{
  int s = 45;
  int t = foo(1, 2, 3) + 12;
  return t + s;
}
```

Second, I examined the assembly code as a baseline in which I could use to see if my changes worked.
```
|   main ();
|           ; var int local_2ch @ rsp+0x2c
|           ; var int local_30h @ rsp+0x30
|           ; var int local_34h @ rsp+0x34
|              ; CALL XREF from 0x140001290 (main + 576)
|           0x140001050      4883ec38       sub rsp, 0x38              ; '8'
|           0x140001054      c74424340000.  mov dword [local_34h], 0
|           0x14000105c      c74424302d00.  mov dword [local_30h], 0x2d ; '-' ; [0x2d:4]=-1 ; 45
|           0x140001064      b901000000     mov ecx, 1
|           0x140001069      ba02000000     mov edx, 2
|           0x14000106e      41b803000000   mov r8d, 3
|           0x140001074      e887ffffff     call fcn.140001000
|           0x140001079      83c00c         add eax, 0xc
|           0x14000107c      8944242c       mov dword [local_2ch], eax
|           0x140001080      8b44242c       mov eax, dword [local_2ch] ; [0x2c:4]=-1 ; ',' ; 44
|           0x140001084      8b4c2430       mov ecx, dword [local_30h] ; [0x30:4]=-1 ; '0' ; 48
|           0x140001088      01c8           add eax, ecx
|           0x14000108a      4883c438       add rsp, 0x38              ; '8'
\           0x14000108e      c3             ret
```

Now, I went through the source code for hours, adding debug statements (which are basically strings throughout my code since LLVM library source lives outside the visual studio solution... this bothers me greatly) until I found the following file [`SelectionDagBuilder.h`](http://llvm.org/doxygen/SelectionDAGBuilder_8h_source.html).
Here, I changed the following line of code
```
private:
  // These all get lowered before this pass.
  void visitInvoke(const InvokeInst &I);
  void visitResume(const ResumeInst &I);

  void visitBinary(const User &I, unsigned Opcode);
  void visitShift(const User &I, unsigned Opcode);
  void visitAdd(const User &I) { visitBinary(I, ISD::SUB); } // Changed to SUB instruction
  void visitFAdd(const User &I) { visitBinary(I, ISD::FADD); }
```

Thus, what I have done is changed the way the DAG is handled when that particular ADD node is visited. This should work...but it didn't. Here lies the actual problem. My assembly code after this change was turning out the exact same. Many bottles of mountain dew were sacrificed as I attempted to understand what was happening. Was my understanding of the backend completely wrong? Why wasn't the add instruction actually being handled? The `visit()` function seemingly never got called for the ADD node... why? Using the LLVM tools I was able to see that the LLVM bytecode contained the add instruction. 
```
define dso_local i32 @main() #1 {
entry:
  %retval = alloca i32, align 4
  %s = alloca i32, align 4
  %t = alloca i32, align 4
  store i32 0, i32* %retval, align 4
  store i32 45, i32* %s, align 4
  %call = call i32 @"?foo@@YAHHHH@Z"(i32 1, i32 2, i32 3)
  %add = add nsw i32 %call, 12
  store i32 %add, i32* %t, align 4
  %0 = load i32, i32* %t, align 4
  %1 = load i32, i32* %s, align 4
  %add1 = add nsw i32 %0, %1
  ret i32 %add1
}
```
So it is in fact there when it emits the LLVM bytecode. I'll spare you hours of reading: I then found out about a cool little tool that actually lets you see the initial DAG before it gets lowered by running the llc tool.

yields

<p align="center">
  <img src ="https://i.imgur.com/qCvylvc.png" />
</p>

As you can see, there's only one node. You would expect there to be many. I then manually printed out the initial SelectionDag by uncommenting a debug line in the source code to see the following.
```
Initial selection DAG: %bb.0 'main:entry'
SelectionDAG has 1 nodes:
  t0: ch = EntryToken
```
Again, only one node. 


# The solution
After what felt like forever, I discovered the following line of source code in [SelectionDagISel.cpp](https://github.com/llvm-mirror/llvm/blob/master/lib/CodeGen/SelectionDAG/SelectionDAGISel.cpp)
```
  if (TM.Options.EnableFastISel) {
    LLVM_DEBUG(dbgs() << "Enabling fast-isel\n");
    FastIS = TLI->createFastISel(*FuncInfo, LibInfo);
  }
```
Long story short: There's something called Fast Instruction Selection which somehow alters the way the the SelectionDag is constructed. aparently, it generates instructions without the use of IR when it seems faster

```"This is a fast-path instruction selection class that generates poor code and doesn't support illegal types or non-trivial lowering, but runs quickly."```

I altered the source code so the if statement always evaluated to false and I then see what I originally expected. The DAG then becomes
```
Initial selection DAG: %bb.0 'main:entry'
SelectionDAG has 38 nodes:
  t3: i64 = Constant<0>
  t9: i64 = GlobalAddress<i32 (i32, i32, i32)* @"?foo@@YAHHHH@Z"> 0
          t0: ch = EntryToken
        t5: ch = store<(store 4 into %ir.retval)> t0, Constant:i32<0>, FrameIndex:i64<0>, undef:i64
      t8: ch = store<(store 4 into %ir.s)> t5, Constant:i32<45>, FrameIndex:i64<1>, undef:i64
    t15: ch,glue = callseq_start t8, TargetConstant:i64<32>, TargetConstant:i64<0>
  t17: ch,glue = CopyToReg t15, Register:i32 $ecx, Constant:i32<1>
  t19: ch,glue = CopyToReg t17, Register:i32 $edx, Constant:i32<2>, t17:1
  t21: ch,glue = CopyToReg t19, Register:i32 $r8d, Constant:i32<3>, t19:1
  t24: ch,glue = X86ISD::CALL t21, TargetGlobalAddress:i64<i32 (i32, i32, i32)* @"?foo@@YAHHHH@Z"> 0, Register:i32 $ecx, Register:i32 $edx, Register:i32 $r8d, RegisterMask:Untyped, t21:1
  t25: ch,glue = callseq_end t24, TargetConstant:i64<32>, TargetConstant:i64<0>, t24:1
  t27: i32,ch,glue = CopyFromReg t25, Register:i32 $eax, t25:1
    t29: i32 = add nsw t27, Constant:i32<12>
  t31: ch = store<(store 4 into %ir.t)> t27:1, t29, FrameIndex:i64<2>, undef:i64
      t32: i32,ch = load<(dereferenceable load 4 from %ir.t)> t31, FrameIndex:i64<2>, undef:i64
      t33: i32,ch = load<(dereferenceable load 4 from %ir.s)> t31, FrameIndex:i64<1>, undef:i64
    t34: i32 = add nsw t32, t33
  t36: ch,glue = CopyToReg t31, Register:i32 $eax, t34
  t37: ch = X86ISD::RET_FLAG t36, TargetConstant:i32<0>, Register:i32 $eax, t36:1
```
As you can see, the DAG now has the correct add token (among many other tokens). Thus, if I go back and re-alter the `visitBinary()` call that I outlined above, I get the expected assembly code! Success, we have just successfully "swapped" a machine instruction
```
|   main ();
|           ; var int local_2ch @ rsp+0x2c
|           ; var int local_30h @ rsp+0x30
|           ; var int local_34h @ rsp+0x34
|              ; CALL XREF from 0x140001290 (main + 576)
|           0x140001050      4883ec38       sub rsp, 0x38              ; '8'
|           0x140001054      c74424340000.  mov dword [local_34h], 0
|           0x14000105c      c74424302d00.  mov dword [local_30h], 0x2d ; '-' ; [0x2d:4]=-1 ; 45
|           0x140001064      b901000000     mov ecx, 1
|           0x140001069      ba02000000     mov edx, 2
|           0x14000106e      41b803000000   mov r8d, 3
|           0x140001074      e887ffffff     call fcn.140001000
|           0x140001079      83c0f4         add eax, -0xc
|           0x14000107c      8944242c       mov dword [local_2ch], eax
|           0x140001080      8b44242c       mov eax, dword [local_2ch] ; [0x2c:4]=-1 ; ',' ; 44
|           0x140001084      8b4c2430       mov ecx, dword [local_30h] ; [0x30:4]=-1 ; '0' ; 48
|           0x140001088      29c8           sub eax, ecx
|           0x14000108a      4883c438       add rsp, 0x38              ; '8'
\           0x14000108e      c3             ret
```

The result of the program is now -57 as you'd expect when you replace addition with subtraction (remembering that all instructions are integer operands, thus foo(1,2,3) = 0)


# Where I Go From Here
It appears that the clang driver is enabling fast instruction selection by default. As of right now, I don't truly know what this means or how it affects backend code generation. There's clearly some value in it, but for now, it simply exists as the flag I've manually disabled to build a full DAG IR graph. I now have a much better understanding of the LLVM backend, and going forward I'm going to be focusing primarily on return instructions. Thus, this is what I'll be focusing on the next few weeks.
- Figure out what FastISel is. I'm going to send an e-mail tomorrow to one of the experienced developers and hopefully get some help here.
- Figure out where the return instruction fits into all this. How would we go about altering the ret instruction?
- Figure out if we should disable the FastISel flag
- Continue to research ways to make ROP gadgets harder to find so that when the time comes, we have a plan of implementation.

# Sources
 - [FastISel Class Reference](http://llvm.org/doxygen/classllvm_1_1FastISel.html)
 - [Life of an instruction in LLVM - Eli Bendersky](https://eli.thegreenplace.net/2012/11/24/life-of-an-instruction-in-llvm)
 - [TableGen Documentation](https://llvm.org/docs/TableGen/index.html)
 - [The Life of a Function - Roel Jordans](http://www.es.ele.tue.nl/~rjordans/5LIM0/03-instruction-selection-scheduling-RA.pdf)
 - [LLVM Programmers Manual](https://llvm.org/docs/ProgrammersManual.html)
 - [A Deeper Look into the LLVM code generator](https://eli.thegreenplace.net/2013/02/25/a-deeper-look-into-the-llvm-code-generator-part-1)
 - [Code Generator Instruction Selection Documentation](https://llvm.org/docs/CodeGenerator.html#instruction-selection)
 
