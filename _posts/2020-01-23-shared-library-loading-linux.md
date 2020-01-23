---
layout: post
title: Linux shared library loading walkthrough
---

This post aims at illustrating loading shared libraries on Linux, specifically
running shared library functions. The problem is that compiler can't know in 
advance at which address shared library will be loaded, and therefore can't 
properly generate function calling code. And the main idea for common solution 
is that calling .so functions is performed through an intermediary table inside 
binary ELF file, which is empty at compile-time but is filled with proper values 
at runtime by dynamic linker.

Let's take a hello world C program:
<pre><code>
#include &lt;stdio.h&gt;
int main(void)
{
	printf("hello!\n");

	return 0;
}
</code></pre>
and build it.

Somewhat suprisingly, by default gcc builds binaries as a shared objects, which
can be seen via `file` utility (see [this StackOverflow post](https://stackoverflow.com/a/34522357) for details).
For the purpose of this post, it's more convenient to build the above program
as executable, so:

`gcc -no-pie hell.c -o hello`

Let's determine how printf is being called and start with ELF header:
<pre><code>
readelf -h hello
...
Entry point address:               <b>0x401040</b>
00400000-00401000 r--p 00000000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/hello
<b>00401000-00402000</b> r-xp 00001000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/hello
00402000-00403000 r--p 00002000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/hello
00403000-00404000 r--p 00002000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/hello
00404000-00405000 rw-p 00003000 fe:02 4296254 /home/ivan/dev/xgi/abi/so/hello
...
</code></pre>

Let's disassemble relevant parts:
<pre><code>
objdump -M intel -d hello
Disassembly of section .text:

<b>0000000000401040</b> &lt;_start&gt;:
<b>0000000000401126</b> &lt;main&gt;:
  401126:	55                   	push   rbp
  401127:	48 89 e5             	mov    rbp,rsp
  40112a:	48 8d 3d d3 0e 00 00 	lea    rdi,[rip+0xed3]        # 402004 &lt;_IO_stdin_used+0x4&gt;
  401131:	e8 fa fe ff ff       	<b>call   401030 &lt;puts@plt&gt;</b>
...
0000000000401030 &lt;puts@plt&gt;:
  <b>401030</b>:	ff 25 e2 2f 00 00    	jmp    QWORD PTR [rip+0x2fe2]        # <b>404018</b> &lt;puts@GLIBC_2.2.5&gt;
...
Relocation section '.rela.plt' at offset 0x4b8 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
<b>000000404018</b>  000200000007 R_X86_64_JUMP_SLO <b>0000000000000000</b> puts@GLIBC_2.2.5 + 0
</code></pre>

Some takeaways:
- entry point address and `main` differ, which is expected since `main` needs a certain environment
  to exist (stack setup, arguments and environment set, etc).
  So entry point is really `_start`, which does housekeeping needed for `main` and jumps to it
- calling external library function (`puts` from libc in this case) is done through so-called 
  PLT ([see this SO answer for details](https://reverseengineering.stackexchange.com/questions/1992/what-is-plt-got/1993#1993))
- PLT entry is empty at compile time (see 'Sym. Value' column)
- glibc symbols are now versioned (which wasn't always the case)

As a side-note, `puts` is contained in both global symbol table (.dynsym) and full symbol table (.symtab):
<pre><code>
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
</code></pre>

So let's check how/when PLT entry gets populated via gdb.

gdb before starting the program:
<pre><code>
(gdb) x /20b 0x404018
0x404018 &lt;puts@got.plt&gt;:	<b>0x36	0x10	0x40	0x00	0x00	0x00	0x00	0x00</b>
0x404020:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x404028:	0x00	0x00	0x00	0x00
</code></pre>

gdb after puts is called:
<pre><code>
(gdb) x /20b 0x404018
0x404018 &lt;puts@got.plt&gt;:	<b>0x80	0x30	0xe4	0xf7	0xff	0x7f	0x00	0x00</b>
0x404020:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x00
0x404028:	0x00	0x00	0x00	0x00
</code></pre>

If we were about to set up a watchpoint on corresponding address, it would trigger
inside `dl_fixup` function which actually does the job.
