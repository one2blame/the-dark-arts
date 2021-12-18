# Fastbin Dup

While the Fastbin Dup technique can be implemented using a heap buffer overflow
, the most common example used to demonstrate this technique is by using the
Use-after-free (UAF) and Double Free vulnerabilities.

## The technique

The steps to execute the Fastbin Dup technique are as follows:

* Leverage a Double Free vulnerability to free a victim chunk twice - this
chunk must be small enough that it gets linked into the fastbin.
* Coerce the program to execute `malloc()` to allocate the victim chunk from
the fastbin.
* Overwrite the `fd` pointer of the victim chunk to point to a fake chunk that
overlaps the location of our arbitrary write target.
  * **Since the inception of this technique, there have been some `glibc`
mitigations implemented to check if the next chunk in the fastbin contains a
valid size field. You can find fake chunk candidates near your target write
location using the `find_fake_fast` command with `pwndbg`.**
* Coerce the program to execute `malloc()` until we receive a pointer to the
same victim chunk, however, this time we don't need to do anything with the
user data. The chunk overlapping our write target now exists within the
fastbin.
* Coerce the program to execute `malloc()` one last time, providing us a
pointer to the chunk overlapping our write target.
* Use the pointer to the fake chunk to conduct our arbitrary write.

### Write targets

So what targets do we wish to write to? Well, the usual candidates are: the
`__free_hook`, `__malloc_hook`, or the Global Offset Table (GOT). Other
techniques also target the stack which is a viable option if you can accurately
determine where the return address of the current function is located. After we
gain the ability to conduct an arbitrary write, gaining code execution should
be trivial with any other technique.

### Requirements

An attacker using the Fastbin Dup must have these conditions present:

* Have chunks that are in the fastbin
* Have a UAF vulnerability with the ability to control the data written to the
free chunk
* Have the ability to control the data written to the chunk that overlaps the
write target (`__free_hook`, etc.)
* Have a memory leak that allows the attacker to defeat ASLR if enabled

## Patch

Unfortunately, the researchers at **Checkpoint Research** proposed protections
for the `fd` pointers used to implement the linked list data structures of the
fastbin and tcache. [[2]](#references) To prevent attackers from leveraging the
Fastbin and Tcache Dup techniques, these researchers implemented the
`PROTECT_PTR` and `REVEAL_PTR` macros for the `fd` pointer of the fastbin and
tcache singly-linked lists.

Summarizing the implementation of these macros, the `fd` pointer of the fastbin
and tcache are mangled using the random bits of the virtual memory address
currently holding the `fd` pointer. The macro definitions are as follows
[[3]](#references):

```c
#define PROTECT_PTR(pos, ptr) \
  ((__typeof (ptr)) ((((size_t) pos) >> 12) ^ ((size_t) ptr)))
#define REVEAL_PTR(ptr)  PROTECT_PTR (&ptr, ptr)
```

These mitigations make the Fastbin Dup technique significantly harder to pull
off. Now, in order to forge a `fd` pointer, the attacker has to leak
sensitive information from the process to derive the location of their
arbitrary write target in memory. They must use that same address to mangle
their forged `fd` pointer before writing it to the victim chunk. The final
nail in the coffin, however, is the fact that the pointer must be page-aligned.
This, coupled with checks done by `glibc` to verify the size field of our fake
chunk overlapping our write target, invalidates our ability to use the above
technique to overwrite something like the `__malloc_hook` to gain control of
the `RIP`.

## References

1. [https://github.com/shellphish/how2heap/blob/master/glibc_2.31/fastbin_dup.c](https://github.com/shellphish/how2heap/blob/master/glibc_2.31/fastbin_dup.c)
2. [https://research.checkpoint.com/2020/safe-linking-eliminating-a-20-year-old-malloc-exploit-primitive/](https://research.checkpoint.com/2020/safe-linking-eliminating-a-20-year-old-malloc-exploit-primitive/)
3. [https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=a1a486d70ebcc47a686ff5846875eacad0940e41](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=a1a486d70ebcc47a686ff5846875eacad0940e41) 
