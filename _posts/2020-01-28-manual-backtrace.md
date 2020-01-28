---
layout: post
title: How function backtrace is generated
---

Let's walk through a backtrace for a "Hello, world!" program, specifically
simpler case with enabled frame pointers. Frame pointer is effectively an
optional pointer, pointing to a portion of the stack dedicated to currently
executing function. For ARM architecture, it occupies register r11.

<pre><code>
int f1(int i) {
	return f2(i);
}

int main(void)
{
	return f1(3);
}
</code></pre>

To enforce usage of frame pointers, -fno-omit-frame-pointer should be used:
<pre><code>
gcc backtrace.c -fno-omit-frame-pointer -o backtrace
</code></pre>
Omitting frame pointer is a performance optimization that makes r11 available
for other uses.

Now to the backtrace itself: the idea is that with frame pointers enabled,
there's effectively a stack-based linked list of two-member structures,
consisting of previous frame pointer and link register (pointer to caller).

Let's stop at `main`, before calling `f1`:
<pre><code>
0x1045c &lt;<main+8&gt;>                mov    r0, #3
0x10460 &lt;<main+12&gt;>               bl     0x1042c &lt;<f1&gt;>

(gdb) i r r11 lr
r11            <b>0x7efff5dc</b>
lr             0x76e8f678
</code></pre>

Now let's make a few more steps and stop exactly after prologue of `f1`:
<pre><code>
0x1042c &lt;<f1&gt;>                    <b>push   {r11, lr}</b>
0x10430 &lt;<f1+4&gt;>                  <b>add    r11, sp, #4</b>
0x10434 &lt;<f1+8&gt;>                  sub    sp, sp, #8 
</code></pre>

Two instructions in bold effectively add a node to the linked list. So that
at any instruction inside `f1`, we could check the previous node link by
looking at `r11`:
<pre><code>
r11            <b>0x7efff5d4</b>
</code></pre>

.. and examining a node that it points to:
<pre><code>
(gdb) x/2x <b>0x7efff5d0</b>
0x7efff5d0:     <b>0x7efff5dc</b>      0x00010464
</code></pre>
(not sure why it points to the second member of the struct, that is where
4 bytes offset comes from, probably some ARM peculiarity)

Note that 0x7efff5dc is a previous value of FP during `main`, and 
0x00010464 is an instruction inside `main` (can be seen by looking at `main`
instruction addresses above).
With this method, entire callstack can be recursively determined.

Determining backtrace without frame pointers is more complicated and generally
less reliable. See [this yosefk post](https://yosefk.com/blog/getting-the-call-stack-without-a-frame-pointer.html) for details.
