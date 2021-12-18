# House of Force

This is a pretty straight-foward and easy technique to understand. To utilize
the **House of Force** technique to gain code execution, an attacker needs the
following [[1]](#references):

* Ability to conduct a heap overflow to overwrite the `top chunk` size field.
* Conduct a `malloc()` call with an attacker-controlled size.
* Conduct a final `malloc()` call where the attacker controls the user data.

And the **House of Force** technique follows these steps:

* Attacker uses a heap buffer overflow to overwrite the `top chunk` size field,
usually something like `0xffffffffffffffff` == `-1`.
  * Because the `top chunk` size field is now `ULLONG_MAX`, calls to `malloc()`
can be arbitrarily large and the `top chunk` will be offset by the size of the
call, allowing the attacker to place the `top chunk` pointer anywhere in memory
.
* Attacker requests a chunk of size `x`, placing the `top chunk` directly
before the target that the attacker intends to write to.
* Attacker uses a final allocation to create a chunk with its user data
overlapping the target.

## What are some good targets to overwrite?

Typically, attackers will attempt to write to these `glibc` symbols:

* `__malloc_hook`
* `__free_hook`

The `__malloc_hook` and `__free_hook` symbols are used by `glibc` to allow
programmers the ability to register functions that will be executed when calls
to `malloc()` or `free()` are made. These can be used by a programmer to
acquire statistics, install their own versions of these functions, etc.

Attackers use these hooks to redirect program execution to memory that they
control within the program. For instance, an attacker could write the address
of their ROP chain into either hook, force the program to call `malloc()` or
`free()`, and then begin executing their ROP chain to conduct a stack pivot,
etc. It's also possible to overwrite either hook to point to a `one_gadget` or
`system()` - you get the picture.

Some other less common targets are:

* The `got.plt` section of the program
* The process stack

Going after the Global Offset Table (GOT) to obtain arbitrary code execution is
only a viable option if full Relocation Read-Only (`RELRO`) [[2]](#references)
is not enabled. Otherwise, the attacker would encounter a `SIGSEV` by trying to
write to memory that is not marked as writeable.

Going after the process stack is also a possibility - the attacker could try to
overwrite the `return address` of a stack frame if they know where it resides
in memory. Unfortunately, Address Space Layout Randomization (`ASLR`) makes this
difficult to achieve.

## Patch

The **House of Force** went unpatched for 13 years, but an update was finally
made to `glibc` to check the size of the top chunk on 16 AUG 2018.
[[3]](#references) The **House of Force** still works for `glibc` versions
less than `2.29`.

## References

1. [https://gbmaster.wordpress.com/2015/06/28/x86-exploitation-101-house-of-force-jedi-overflow/](https://gbmaster.wordpress.com/2015/06/28/x86-exploitation-101-house-of-force-jedi-overflow/)
2. [https://ctf101.org/binary-exploitation/relocation-read-only/](https://ctf101.org/binary-exploitation/relocation-read-only/)
3. [https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=30a17d8c95fbfb15c52d1115803b63aaa73a285c](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=30a17d8c95fbfb15c52d1115803b63aaa73a285c)
