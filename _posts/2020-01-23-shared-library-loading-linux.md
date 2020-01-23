---
layout: post
title: Linux shared library loading walkthrough
---

This post aims at illustrating loading shared libraries on Linux, specifically
running shared library functions. The main idea behind it is that
calling .so functions is performed through an intermediary table inside binary
ELF file, which is empty at compile-time but is filled with proper values at
runtime by dynamic linker.

Let's take a hello world C program:
<pre>
#include <stdio.h>
int main(void)
{
	printf("hello!\n");

	return 0;
}
</pre>
and build it.

Suprisingly enough, by default gcc builds binaries as a shared objects, which
can be seen by `file` utility. See [this StackOverflow post](https://stackoverflow.com/a/34522357) for details.
For the purpose of this post, it's more convenient to build the above program
as executable, so:
`gcc -no-pie main.c -o main`

Let's determine how printf is being called and start with main:
<pre>
readelf -h main
...
Entry point address:               <b>0x401040</b>
00400000-00401000 r--p 00000000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
<b>00401000-00402000</b> r-xp 00001000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
00402000-00403000 r--p 00002000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
00403000-00404000 r--p 00002000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
00404000-00405000 rw-p 00003000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/main
...
</pre>

Let's disassemble main:
<pre>
objdump -M intel -d main
Disassembly of section .text:

0000000000401040 <_start>:
<b>0000000000401126</b> <main>:
  401126:	55                   	push   rbp
  401127:	48 89 e5             	mov    rbp,rsp
  40112a:	48 8d 3d d3 0e 00 00 	lea    rdi,[rip+0xed3]        # 402004 <_IO_stdin_used+0x4>
  401131:	e8 fa fe ff ff       	<b>call   401030 <puts@plt></b>
...
0000000000401030 <puts@plt>:
  401030:	ff 25 e2 2f 00 00    	jmp    QWORD PTR [rip+0x2fe2]        # <b>404018</b> <puts@GLIBC_2.2.5>
...
Relocation section '.rela.plt' at offset 0x4b8 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
<b>000000404018</b>  000200000007 R_X86_64_JUMP_SLO <b>0000000000000000</b> puts@GLIBC_2.2.5 + 0
</pre>
Some takeaways:
- entry point address and main differ, which is expected since main needs a certain environment
  to exist (stack setup, arguments and environment set, etc).
  So entry point is really `_start`, which does housekeeping needed for `main` and jumps to it.
- calling external library function (puts from libc in this case) is done through so-called 
  PLT ([see this SO answer for details](https://reverseengineering.stackexchange.com/questions/1992/what-is-plt-got/1993#1993))
- PLT entry is empty at compile time (see 'Sym. Value' column)
- glibc symbols are now versioned (wasn't always the case)

As a side-note, `puts` is contained in both global symbol table (.dynsym) and full symbol table (.symtab):
<pre>
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
</pre>

So let's check how/when PLT entry gets populated via gdb.

gdb before starting the program:
<pre>
(gdb) x /20b 0x404018
0x404018 <puts@got.plt>:	<b>0x36	0x10	0x40	0x00	0x00	0x00	0x00	0x00</b>
0x404020:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x404028:	0x00	0x00	0x00	0x00
</pre>

gdb after puts is called:
<pre>
(gdb) x /20b 0x404018
0x404018 <puts@got.plt>:	<b>0x80	0x30	0xe4	0xf7	0xff	0x7f	0x00	0x00</b>
0x404020:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x404028:	0x00	0x00	0x00	0x00
</pre>

Note: if we were about to set up a watchpoint on corresponding address, it would trigger
inside `dl_fixup` function which actually does the job.
