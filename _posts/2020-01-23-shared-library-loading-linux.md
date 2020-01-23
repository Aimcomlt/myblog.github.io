---
layout: post
title: Linux shared library loading walkthrough
---

This post aims to illustrate loading shared libraries on Linux, specifically
loading and running shared library functions. The main idea behind it is that
calling .so functions is performed through an intermediary table, which is 
empty at compile-time but is filled with proper values at loading time by dynamic
linker.

Let's take a hello world C program:
```c
#include <stdio.h>
int main(void)
{
	printf("hello!\n");

	return 0;
}
```
and build it.

Suprisingly enough, by default gcc builds binaries as a shared objects, which
can be seen by `file` utility. See [this StackOverflow link](https://stackoverflow.com/a/34522357) for details.
For the purpose of this post, it's more convenient to build the above program
as executable, so:
`gcc -no-pie main.c -o main`

Let's start with how printf is being called:
```
readelf -h main
...
Entry point address:               **0x401040**
00400000-00401000 r--p 00000000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
**00401000-00402000** r-xp 00001000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
00402000-00403000 r--p 00002000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
00403000-00404000 r--p 00002000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
00404000-00405000 rw-p 00003000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
...
```

Let's disassemble main:
```
objdump -M intel -d main
Disassembly of section .text:

0000000000401040 <_start>:
***0000000000401126*** <main>:
  401126:	55                   	push   rbp
  401127:	48 89 e5             	mov    rbp,rsp
  40112a:	48 8d 3d d3 0e 00 00 	lea    rdi,[rip+0xed3]        # 402004 <_IO_stdin_used+0x4>
  401131:	e8 fa fe ff ff       	**call   401030 <puts@plt>**
...
0000000000401030 <puts@plt>:
  401030:	ff 25 e2 2f 00 00    	jmp    QWORD PTR [rip+0x2fe2]        # **404018** <puts@GLIBC_2.2.5>
...
Relocation section '.rela.plt' at offset 0x4b8 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
**000000404018**  000200000007 R_X86_64_JUMP_SLO **0000000000000000** puts@GLIBC_2.2.5 + 0
```
Some takeaways:
- entry point address and main differ, which is expected since main needs a certain environment
  to exist (stack setup, arguments and environment set, etc).
  So entry point is really `_start`, which does housekeeping needed for `main` and jumps to it.
- calling external library function (puts from libc in this case) is done through so-called 
  PLT ([see this SO answer for details](https://reverseengineering.stackexchange.com/questions/1992/what-is-plt-got/1993#1993))
- PLT entry is empty at compile time (see 'Sym. Value' column)
- glibc symbols are now versioned (wasn't always the case)

As a side-note, `puts` is contained in both global symbol table (.dynsym) and full symbol table (.symtab):
```
Symbol table '.dynsym' contains 6 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND _ITM_deregisterTMCloneTab
     2: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@GLIBC_2.2.5 (2)

Symbol table '.symtab' contains 66 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
.......
    48: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND puts@@GLIBC_2.2.5
```

So let's check if/when PLT entry gets populated via gdb.

gdb at the beginning:
```
(gdb) x /20b 0x404018
0x404018 <puts@got.plt>:	**0x36	0x10	0x40	0x00	0x00	0x00	0x00	0x00**
0x404020:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x404028:	0x00	0x00	0x00	0x00
```

gdb after puts is called:
```
(gdb) x /20b 0x404018
0x404018 <puts@got.plt>:	**0x80	0x30	0xe4	0xf7	0xff	0x7f	0x00	0x00**
0x404020:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x404028:	0x00	0x00	0x00	0x00
```

Note: if we were about to set up a watchpoint on corresponding address, it would trigger
inside `dl_fixup` function.
