# Weeks Of 11/5 Research Update
This week, I spent a lot of time just building the open source code (and learning some things along the way). Previously, I had always used internal SIE methods to build any version of LLVM/clang needed. I had never actually gone through the process of building the opensource LLVM trunk build. This led to some problems which I had to work around which I'll document below. After successfully building clang in a Windows environment and generating the correct visual studio solutions, I began to investigate clang source code itself with the idea that I'll soon be looking into the pieces that do the backend code generation.

#Building LLVM for Windows/Visual Studio
What a nightmare. If you have the option, I recommend just building opensource clang using GCC for a Linux environment. Things will go much smoother, and you won't have to worry about standard library compatibility, visual studio solutions, host environment mismatches, and all the other headaches that accompany LLVM in Windows environments. After finally setting up LLVM to build correctly, I realized the entire build process takes about ~2 hours to complete on my 8 core machine. Usually, I'm in an environment where I can use distributed build systems so I didn't account for this earlier. Turns out, LLVM is big. Feel free to read more about SIE's distributed build system [here](https://www.snsystems.com/tech-blog/2014/01/06/building-with-the-network/).

<p align="center">
  <img src ="https://i.imgur.com/ik7zcP6.png" />
</p>


I'm at the point now where I'll be continuously be building LLVM (mainly just clang and the clang driver), so I'm going to document the steps needed to successfully build the binaries so I can avoid any headaches in the future.

1.) Download the source repo(s) using SVN. I've found that using the GitHub LLVM mirror causes problems with universal line endings causing tests to fail, as well as revision history being a bit confusing. For this reason you should use SVN... as much as I dislike it. There is a plan for an official LLVM GitHub migration in the next year or so, but alas we are stuck with SVN. The following [link](https://clang.llvm.org/get_started.html) list some of the relevant endpoints: 
- LLVM Trunk -> http://llvm.org/svn/llvm-project/llvm/trunk
- Clang -> http://llvm.org/svn/llvm-project/cfe/trunk
- Clang Tools -> http://llvm.org/svn/llvm-project/clang-tools-extra/trunk
- Compiler-RT -> http://llvm.org/svn/llvm-project/compiler-rt/trunk

Note that I am currently using revision `346633 @ trunk` and will probably continue to work with this revision throughout the projects lifetime. I don't see any reason to keep up to date with trunk but never say never.

2.) Download [CMake](https://cmake.org/download/) for Windows and add the executable to your system path

3.) Make sure you have an instance of VS or VS build tools installed, and make sure the visual studio compiler `cl.exe` is on your system path. This is the compiler which will build LLVM

4.) Either open a VS dev prompt which initializes all variables or call the corresponding file `vcvars64.bat` from an administrator command prompt. This will set up the corresponding variables within that prompt's path, making it so you can find dependencies. (This makes it so you can find necessary runtime libraries and DLL's when building clang)

5.) Add python to your system path (used for LIT test among other things)

6.) Run the following command from wherever you want the build output to be placed
```
cmake -G "Visual Studio 15 2017" -A x64 -Thost=x64 <PATH_TO_LLVM>\llvm
```
This will generate the VS 2017 `.sln` build files and all the source in the desired folder. It's important to have the `-A x64` and `Thost=x64` to make the solutions 64 bit compatible. If you don't the binaries will be built in 32 bit mode and things will fail to link together... I learned this the hard way.

7.) Finally, open the `LLVM.sln` file and build the `BUILD_ALL` project. This will take a while. The output(binaries) will be placed in the `Debug` folder assuming that's the configuration you built with.

At this point, you should test your binaries. Create a simple hello world CPP program and run it through the clang driver, but make sure you use `clang-cl` like so:
```
clang-cl hello-world.cpp
```

you HAVE to use clang-cl if you want to link against the MSVC stdlib and be completely compatible with Windows and Visual Studio's `cl.exe`. This is new to me, as I primarily build executables that run in a FreeBSD environment (PS4).  While debugging linker errors, I stumbled upon the following quote from [Mozilla](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Build_Instructions/Building_Firefox_on_Windows_with_clang-cl) which summed it up well: "clang-cl is a new compiler from the LLVM project that attempts to be a drop-in replacement for MSVC's cl.exe. It also brings the full power of the LLVM toolchain to Windows."


#Diving into Clang Source
I dove into the clang source, first investigating the driver source. I was looking particularly for the pieces of source code that handle back-end code generation, being that they are of interest in preventing ROP gadgets detectability. After setting the correct startup project, adjusting the compiler flags in Visual Studio, and running in debug mode, I was able to see exactly what the compiler driver was doing.

<p align="center">
  <img src ="https://i.imgur.com/b0mcmTH.png" />
</p>


When compiling a simple `hello_world.cpp` source file, the following commands were actually being executed:
```

 "clang.exe" -cc1 -triple x86_64-pc-windows-msvc19.10.25019 -emit-obj -mrelax-all -mincremental-linker-compatible -disable-free -main-file-name hello_world.cpp -mrelocation-model pic -pic-level 2 -mthread-model posix -relaxed-aliasing -fmath-errno -masm-verbose -mconstructor-aliases -munwind-tables -target-cpu x86-64 -mllvm -x86-asm-syntax=intel -D_MT -flto-visibility-public-std --dependent-lib=libcmt --dependent-lib=oldnames -stack-protector 2 -fms-volatile -fdiagnostics-format msvc -dwarf-column-info -debugger-tuning=gdb -momit-leaf-frame-pointer -resource-dir "build\\Debug\\lib\\clang\\8.0.0" -internal-isystem "build\\Debug\\lib\\clang\\8.0.0\\include" -internal-isystem "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Tools\\MSVC\\14.10.25017\\include" -internal-isystem "C:\\Program Files (x86)\\Windows Kits\\NETFXSDK\\4.6.1\\include\\um" -internal-isystem "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.15063.0\\ucrt" -internal-isystem "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.15063.0\\shared" -internal-isystem "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.15063.0\\um" -internal-isystem "C:\\Program Files (x86)\\Windows Kits\\10\\include\\10.0.15063.0\\winrt" -fdeprecated-macro -fdebug-compilation-dir "C:\\Users\\justice\\Desktop\\llvm-snv-source\\example_code" -ferror-limit 19 -fmessage-length 317 -fno-use-cxa-atexit -fms-extensions -fms-compatibility -fms-compatibility-version=19.10.25019 -std=c++14 -fdelayed-template-parsing -fobjc-runtime=gcc -fdiagnostics-show-option -fcolor-diagnostics -o "hello_world-b54570.obj" -x c++ hello_world.cpp -faddrsig
 ```
 ```
 "C:\\Program Files (x86)\\Microsoft Visual Studio\\2017\\Community\\VC\\Tools\\MSVC\\14.10.25017\\bin\\HostX64\\x64\\link.exe" -out:hello_world.exe -nologo "hello_world-b54570.obj"
```

You can see the driver simply does the compilation, then the link phase on the corresponding `.obj` file. I still need to dive into at what point in the compilation phase the assembly generation happens, and what source code I should be looking at and stepping through to see backend code generation in action. This is my primary goal for next week.


## For Next Week
- Dive into the Clang back-end code and see where exactly assembly instructions are generated.
- Research techniques for preventing ROP gadgets from detectability (I've heard on using No-op's from a compiler manager at Sony)
- Use the ROP tools mentioned above to analyze some industry executables, maybe a game if I can get my hands on a sample.

## Sources for this week
### Information sources:
- [Clang Documentation: MSVC compatibility](https://clang.llvm.org/docs/MSVCCompatibility.html)
- [What is Clang-cl by Google](https://llvm.org/devmtg/2014-04/PDFs/Talks/clang-cl.pdf)
- [Building Firefox on Windows with clang-cl
](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/Build_Instructions/Building_Firefox_on_Windows_with_clang-cl)
- [Getting Started: Building and Running Clang](https://clang.llvm.org/get_started.html)
- [Building LLVM with CMake](https://llvm.org/docs/CMake.html)
- [Clang Compiler User’s Manual](https://clang.llvm.org/docs/UsersManual.html)
- [Kaleidoscope: Compiling to Object Code](https://llvm.org/docs/tutorial/LangImpl08.html)

### Image Sources
- https://www.snsystems.com/tech-blog/2014/01/06/building-with-the-network/
