# Calling libc functions / syscalls

## Calling conventions

First, let's talk about **calling conventions**. For the System V `x86`
application binary interface (ABI), parameters to functions are passed on the
stack in reverse order such that the first parameter is the last value pushed
to the stack.

Functions are called using the `call` instruction, pushing the address of the
next instruction onto the stack and jumping to the operand. Functions return
to the caller using `ret`, popping a value from the stack and jumping to it.

Function calls preserve the registers `ebx`, `esi`, `edi`, `ebp`, and `esp`.
`eax`, `ecx`, and `edx` are scratch registers, their values might not be the
same after a function call. The return value of a function is stored in `eax`,
unless the return value is a 64-bit value, then the higher 32-bits will be
returned in `edx`.

For the System V `x86-64` ABI, parameters to functions are passed in the
registers `rdi`, `rsi`, `rdx`, `rcx`, `r8`, and `r9`. If a function call
requires more than 6 parameters, further values are passed to the function
call on the stack in reverse order. Functions return to the caller using `ret`,
popping a value from the stack and jumping to it.

Function calls preserve the registers `rbx`, `rsp`, `rbp`, `r12`, `r13`, `r14`,
and `r15`. `rax`, `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`, `r10`, and `r11` are
scratch registers, their values might not be the same after a function call.
The return value of a function is stored in `rax`, unless the return value is a
128-bit value, then the higher 64-bits will be returned in `rdx`.
[[1]](#references)

## The standard C library and syscalls

**What's a `libc` function?**

The term `libc` is shorthand for the "standard C library". The most widely used
C library on Linux is the [GNU C Library](http://www.gnu.org/software/libc/),
referred to as `glibc`. The pathname for your installation of `libc` on a
Linux operating system is probably something similar to: `/lib/libc.so.6`. For
a dynamically-linked target binary, you can use `ldd` to print its shared
object dependencies - you'll usually find `libc.so.6` there.
[[2]](#references)

`libc` functions are functions exported by the `libc.so.6` shared library.
`libc` functions are commonly used functions within the C programming
language, for example: `printf()`, `puts()`, `gets()`

**What's a `syscall`?**

A system call (`syscall`), sometimes called kernel calls, is a request for
service that a program makes of the kernel. This is usually a privileged
operation, like changing permissions of a file, opening a file descriptor, etc.
The `libc` functions mentioned earlier usually handle `syscall`s for the
programmer, however, an attacker sometimes has to make `syscall`s directly for
an exploit to work. [[3]](#references)

To make a `syscall` directly on `x86` on a Linux operating system, first
ensure your registers contain their necessary values.
[[4]](#references) Then, issue the instruction `int 0x80`. This instruction is
an interrupt that passes control to interrupt vector `0x80`. On Linux, this
interrupt vector is used to handle `syscall`s. [[5]](#references)

To make a `syscall` directly on `x86-64` on a Linux operating system, first
ensure your registers contain their necessary values.
[[6]](#references) Then, issue the instruction `syscall`. [[7]](#references)

## Using ROP to call a libc function

Now that we understand the calling conventions, we can use ROP to call `libc`
functions. We're going to assume that the attacker already has control over the
`stack` and the `EIP` or `RIP`. We're also going to assume that the attacker
has used an information leak to expose sensitive information, namely the base
of `libc` within memory. With these conditions, the attacker can execute one of
the most basic ROP attacks: `ret2libc`.

The attacker must use their control over the `EIP`/`RIP` and `stack` to
construct a stack frame that will call `libc`'s `system()` function. The
attacker will also craft the stack frame so that the `system()` function will
use the string `'/bin/sh'` as an argument.

Here's what the stack would look like for a `ret2libc` attack on `x86`:

```python
p32(system)         # Address of system() function within libc
p32(0xcafebabe)     # Fake return pointer
p32(bin_sh_ptr)     # arg0: pointer to '/bin/sh' string
```

After the `stack` is corrupted with the attacker's `ret2libc` `stack` frame,
when the vulnerable function conducts a `ret`, the program will `pop` the
`system()` address from the `stack` and jump to `system()` within `libc`. The
`system()` function will continue execution and use the pointer to the
`'/bin/sh'` string as an argument, executing `system('/bin/sh')`.

For `x86-64`, remember that we need to place our arguments into registers
before making calls to `libc` functions. A `ret2libc` attack will look slightly
different:

```python
p64(pop_rdi_ret)    # pop rdi; ret gadget
p64(bin_sh_ptr)     # arg0: pointer to '/bin/sh' string
p64(system)         # Address of system() function within libc
```

For `x86-64`, `system()` expects `arg0` to be contained within the `rdi`
register. Using our techniques discussed earlier to find `ROP` gadgets, we
find a `pop rdi; ret` gadget within either `libc` or the target binary. We use
this gadget to `pop` the address of our `'/bin/sh'` string into `rdi` and then
we `ret` into the address of `system()`. Just like in `x86`, this executes
`system('/bin/sh')`.

**The next section will cover executing `syscall`s directly.**

## References

1. [https://wiki.osdev.org/System_V_ABI](https://wiki.osdev.org/System_V_ABI)
2. [https://man7.org/linux/man-pages//man7/libc.7.html](https://man7.org/linux/man-pages//man7/libc.7.html)
3. [https://www.gnu.org/software/libc/manual/html_node/System-Calls.html](https://www.gnu.org/software/libc/manual/html_node/System-Calls.html)
4. [https://web.archive.org/web/20200218024630/https://syscalls.kernelgrok.com/](https://web.archive.org/web/20200218024630/https://syscalls.kernelgrok.com/)
5. [https://en.wikipedia.org/wiki/Interrupt_vector_table](https://en.wikipedia.org/wiki/Interrupt_vector_table)
6. [https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/](https://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)
7. [https://www.cs.fsu.edu/~langley/CNT5605/2017-Summer/assembly-example/assembly.html](https://www.cs.fsu.edu/~langley/CNT5605/2017-Summer/assembly-example/assembly.html)
