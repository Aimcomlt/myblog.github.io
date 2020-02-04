---
layout: post
title: On Application Binary Interface
---

Some time ago I tried to dive into the topic of ABI (applicaiton binary interface)
and it turned out to be surprisingly deep and multi-faceted. This post is a 
collection of hopefully key points, and just random ABI-related pieces.

### Overview
Wikipedia definition gives us:
> In computer software, an application binary interface (ABI) is an interface
> between two binary program modules; often, one of these modules is a library or
> operating system facility, and the other is a program that is being run by a
> user. 

[Top SO answer](https://stackoverflow.com/a/2456882) is a bit more specific:
> An ABI is very similar. Think of it as the compiled version of an API (or as an
> API on the machine-language level). When you write source code, you access the
> library through an API. Once the code is compiled, your application accesses
> the binary data in the library through the ABI. The ABI defines the structures
> and methods that your compiled application will use to access the external
> library (just like the API did), only on a lower level.

In general, ABI covers various parts of the stack (from CPU register allocation
and usage, to kernel ABI, to userspace library and language ABI), including:
- CPU instruction sets
- calling conventions (e.g. passing of arguments and return value)
- layout and alignments of data (e.g. in structures)
- system call interface (how exactly userspace can call a kernel function)
- binary format of object files
- language-specific stuff like C++ name mangling and exceptions
- etc etc

### ARM
Speaking about ARM in particular, ABI for 32 bit platforms is covered in about
a [dozen](https://developer.arm.com/architectures/system-architectures/software-standards/abi) of separate documents.

### Linux kernel
Linux kernel ABI is known to be very stable, which in practice means that any
program compiled for an old kernel (let's say 10+ years old) has a very good
chances to run on modern kernel. This is ensured, in particular, by keeping
system call interface stable, so for example type or number of parameters 
never changes, and system calls are never removed.

Note: internal kernel ABI and API is not stable at all, interfaces are often 
refactored and changed for the better, triggering corresponding changes in 
modules that are using them.

### libc
Libc is generally backward-compatible, however nowadays glibc provides versions
for its exported symbols, meaning that program compiled with newer glibc cannot
be run on older version. This is why some people keep old Linux version
on a build server for compiling and linking against old libc.

### Shared libraries
Shared libraries typically use major-minor (sometimes also patch) version scheme,
e.g. libfoo.so.X.Y, with major version changing together with incompatible 
ABI changes, and minor or patch versions chaning with compatible changes/bug
fixes.

Here's a [good article](http://bottomupcs.sourceforge.net/csbu/x4012.htm) on the topic.

Since different distributions contain different sets and versions of libraries,
it's typical for proprietary Linux programs to ship with all of its dependent
libraries, wrapped with `LD_LIBRARY_PATH` scripts.

### Compilers
Compilers are naturally related to ABI, since they actually produce machine code.
And speaking of C/C++, since language standard doesn't enforce any particular
ABI, code produced by two different compiler or two versions of the same compiler
is generally incompatible. Quoting [this SO post](https://softwareengineering.stackexchange.com/questions/235706/are-c-object-files-created-with-different-compilers-binary-compatible/235768#235768):
> This lack of compatibility also extends to different versions of the same
> compiler. In general, programs and libraries compiled with older and newer
> versions of a compiler cannot be linked together, and those compiled with MSVC
> cannot be linked with those compiled by GCC.
> 
> There is a specific and very useful exception. Every platform provides a
> dynamic linking ABI (Application Binary Interface) and any program in any
> language that can conform to that ABI is compatible. Therefore it is generally
> possible to build a DLL (on Windows) with MSVC (or something else) and call it
> from a program compiled by a different version of MSVC or by GCC and vice
> versa.

### Toolchains
Further related to compilers is a notion of toolchain, which usually consists of
compiler, binary utils (e.g. `as` or `ld`), libc, debuggers, etc. Linux toolchains
are usually named using the following scheme:
`arch-vendor-(os-)abi`, for example `arm-none-eabi-gcc`.

Personal anecdote: kernel compiled with a newer version of the same toolchain
silently crashed on start, while using two years older version produced a 
working kernel.
