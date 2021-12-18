# **Unsortedbin Attack**

The **Unsortedbin Attack** and the Use-after-free (UAF) vulnerability are loosely
related enough for me to write about it in this section. The **Unsortedbin Attack**
can be used if you have a heap overflow vulnerability, as shown in the House of
Orange, however, for that version of the technique to work some special
conditions need to be present. For the purpose of this discussion, we're only
going to concern ourselves with the **Unsortedbin Attack** implemented with a
Use-after-free (UAF) vulnerability.

## How it works

This technique is actually pretty simple. To leverage this attack, the attacker
needs to be able to:

* Acquire a chunk on the heap that is too large for the fastbin
* Acquire another chunk after the previously mentioned chunk that acts as a
fencepost between the previously mentioned chunk and the top chunk.
* Free the first chunk.
* Edit the first chunk (UAF vulnerability).
* Execute another `malloc()`, requesting a chunk of the same size as the one
that was `free()`'d and linked into the Unsortedbin.

### So what happens here?

The first chunk on the heap is too large for the fastbin so, when it is
`free()`'d, it is linked into the Unsortedbin. This free chunk will be linked
into a small or largebin, based upon its size, during the next call to
`malloc()`. We create a buffer chunk between this previously mentioned chunk
and the top chunk prior to trying to `free` the first chunk, otherwise our
victim chunk will be consolidated into the top chunk.

With our Use-after-free (UAF) vulnerability, we overwrite the `bk` pointer of
the chunk in the Unsortedbin to point to an address that we want to write to.
What are we writing? Well, when an Unsortedbin chunk is chosen to service a
`malloc()` call, the memory address of the `main_arena` that contains the head
of the Unsortedbin is written to where the `bk` points to.

## How is this used for exploitation?

Honestly, the **Unsortedbin Attack** is often used to enable other techniques. It's
usually a precursor to further exploitation but, by itself, it's not useful for
gaining code execution.

A great example technique that leverage the **Unsortedbin Attack** to gain code
execution is the [House of Orange](./house-of-orange.md). In that example,
however, the vulnerability leveraged to overwrite the `bk` of the chunk in the
Unsortedbin is a heap overflow.

Another good example is the [zerostorage](#references) challenge from 0ctf 2016
. In this challenge, the attacker uses a UAF vulnerability to overwrite the
`bk` of a chunk in the Unsortedbin and then forces an allocation of this chunk
to overwrite the symbol `libc.global_max_fast`. `global_max_fast` represents
the largest size a fastbin chunk can be - overwriting it with the memory
address that contains the Unsortedbin head, making the size of fastbin chunks
almost arbitrarily large.

Using the above, the attacker now has the ability to create fastbin chunks and,
because a UAF vulnerability is present, the attacker can leverage the Fastbin
Dup to gain code execution, targeting the `__free_hook`.

## Patch

The **Unsortedbin Attack** went unpatched until `glibc 2.29`, commited to the
`glibc` repo on 17 AUG 2018. Before the patch, no integrity checks were
conducted on the fake chunk pointed to by the corrupted `bk` pointer used in
this technique. Now, integrity checks are conducted on the free chunk being
inspected in the Unsortedbin, the `next` chunk of the victim free chunk, and
the chunk pointed to by `bk`. [[2]](#references).

## References

1. [https://guyinatuxedo.github.io/31-unsortedbin_attack/0ctf16_zerostorage/index.html](https://guyinatuxedo.github.io/31-unsortedbin_attack/0ctf16_zerostorage/index.html)
2. [https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=b90ddd08f6dd688e651df9ee89ca3a69ff88cd0c](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=b90ddd08f6dd688e651df9ee89ca3a69ff88cd0c)
