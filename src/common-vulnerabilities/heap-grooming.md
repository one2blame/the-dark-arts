# Heap grooming

Let's first define **heap grooming** by identifying that it's not
**heap spraying**. While these two concepts are related, heap spraying is used
to make other exploitation techniques more reliable. Heap spraying gained
popularity starting in 2001, and is primarily used to store code in a reliable
location when attempting to gain code execution. [[1]](#references)

**Heap grooming** or **heap feng shui** or **heap shaping** is a more nuanced
technique that attempts to shape the heap in a manner that allows for follow
on exploitation by the attacker. The use of this technique is pivotal to
improving the reliability of using heap memory corruption for exploitation. For
example, in order to use a heap overflow to create the conditions for
overlapping chunks or a Use-after-free (UAF) vulnerability, the attacker might
want to have chunks of specific sizes in specific locations or have program
data structures in advantageous locations for sensitive information leaks, etc.

## Examples

### Heap Feng Shui in Javascript

First, we'll start with [this BlackHat](#references) talk by Alexander Sotirov.
It's actually pretty awesome that something this intense was discussed as early
as 2007. Alexander's talk begins with a description of heap spraying, but shows
that, while heap spraying is useful, it's still unreliable - we need the heap
to be in a deterministic state to achieve reliable exploitation.

His **Heap Feng Shui** portion of the talk discusses how the heap allocator is
deterministic in nature. The attacker must coerce the program to execute a
series of allocations and frees until he deterministically controls important
locations in the heap. Eventually, he demonstrates that the attacker can place
malicious bytes onto the heap in a repeated nature, coerce the program to free
his allocated chunks, and then, when the program resumes a different
subroutine and allocates the recently `free()`d chunks, the attacker can
exploit an unitialized variable vulnerability to gain code execution.

He goes on to discuss how the memory allocator works for Javascript / Internet
Explorer and his creation of a library that allows him to log, debug, allocate,
and free heap chunks of arbitrary size and contents within the Javascript /
Internet Explorer heap.

### Grooming the iOS Kernel Heap

Next, we'll talk about
[Azeria's discussion on grooming the iOS kernel heap](#references). In this
real world example, she demonstrates the exploitation of a one byte, heap
overflow vulnerability to gain a UAF that allows her to leak sensitive
information from program memory.

In this example, Azeria discusses that the state of the heap before
exploitation is fragmented, with free and allocated chunks sparsely strewn
about the heap. She coerces the application to fill the "gaps" in the heap by
allocating objects until the freelist is empty. Subsequent allocations will
then start at the highest address of the heap where she begins to create a
series of "exploit zones", comprised of victim objects and placeholder objects.
The victim objects will be used to overflow into succeeding objects on the
heap. She `free()`s all of the placeholder objects in quick succession, linking
them into the freelist. When the kernel allocates her target objects that she
wishes to overflow into, the new allocations will be adjacent to the victim
objects.

Using **heap grooming**, Azeria demonstrates that an attacker can coerce the
allocator into placing victim objects next to sensitive kernel objects on the
heap in order to corrupt heap memory, leading to a sensitive information
disclosure.

### Chrome OS exploit: one byte overflow and symlinks

`shill`, the Chrome OS network manager, had a one byte heap overflow in its
DNS library. Attackers could communicate with the library using an HTTP proxy
that was listening on `localhost`. The author demonstrates his use of
**heap grooming** to ensure that the memory layout of the vulnerable process
remained predictable so that he could reliably place allocations in the same
location in heap memory.

The author demonstrates that, upon each connection, four allocations are
conducted by the program to hold program data on the heap. When the connection
closes, however, the middle `0x820` bytes in these four allocations are
`free()`d and consolidated, and the two still allocated chunks protect the
free `0x820` chunk from being further consolidated or modified.

The author goes on to explain that, in order to continue to reliably use this
`0x820` chunk of memory to obtain overlapping chunks, he has to coerce the
program to conduct "tons of small allocations" in order to empty the freelist.
Finally, subsequent allocations allow him to reliably use his `0x820` chunk of
memory, eventually conducting a heap buffer overflow to shrink a free chunk
causing a future `free()` operation on a succeeding chunk to overlap an
allocated chunk. [[4]](#references)

## Malloc tunables

A listing of `malloc` tunables can be found [here](#references). There's not
much documentation on how `malloc` tunables can affect exploitation methods,
or my Google-Fu is probably just not good enough. Either way, in this section
I'll wargame some `malloc` tunable changes that could possibly bring about
interesting changes to how we approach heap exploitation.

### `glibc.malloc.tcache_count` and `glibc.malloc.tache_max`

These two tunables are interesting if we know that the target `glibc` uses
`tcache`s. If I knew that the vulnerable program was vulnerable to a double
free, I would probably go for a Fastbin Dup attack. The default number of
chunks that get linked into the `tcache` is `7`, however, if this was modified
to be something else, we'd have to do some debugging in production conditions
to determine the `tcache_count`. This tunable would affect the number of chunks
I `free()` in order to fill the `tcache` before I attempt to abuse the fastbin.

`tcache_max` could also affect any attempts to go after heap metadata for
chunks linked into the unsortedbin. If `tcache_max`'s size overlapped with
sizes that are usually used for the unsortedbin, I would have to ensure that
the `tcache` is full before attempting to link chunks into the unsortedbin.

### `glibc.malloc.arena_max` and `MALLOC_ARENA_MAX`

The `glibc.malloc.arena_max` tunable overrides anything set by
`MALLOC_ARENA_MAX`, but they achieve the same outcome. These two tunables set
the maximum number of arenas that can be used by the process across all threads
. For 32-bit systems, this value is twice the number of cores online and for
64-bit systems, this value is 8 times the number of cores online.

This is important if we're attempting to exploit the heap of a vulnerable
program that leverages multi-threaded operations. `MALLOC_ARENA_MAX` increases
the number of available memory pools that threads can allocate memory from.
A higher number of arenas helps to prevent thread contention for heap
resources as each thread has to lock the arena before allocating or `free()`ing
memory.

If we have local access to the vulnerable application, we can modify the
`MALLOC_ARENA_MAX` environment variable and set it to `1`. This forces each
thread of our target program to use the same arena, allowing us to possibly
exploit race conditions that might be present, or it could just make grooming
the heap easier. Either way, it's important to keep in mind how many arenas are
present when exploiting a program that leverages multi-threading as this will
guide our approach on how to effectively groom the heap across multiple arenas.

## References

1. [https://seclists.org/bugtraq/2001/Jul/533](https://seclists.org/bugtraq/2001/Jul/533)
2. [https://www.blackhat.com/presentations/bh-europe-07/Sotirov/Presentation/bh-eu-07-sotirov-apr19.pdf](https://www.blackhat.com/presentations/bh-europe-07/Sotirov/Presentation/bh-eu-07-sotirov-apr19.pdf)
3. [https://azeria-labs.com/grooming-the-ios-kernel-heap/](https://azeria-labs.com/grooming-the-ios-kernel-heap/)
4. [https://googleprojectzero.blogspot.com/2016/12/chrome-os-exploit-one-byte-overflow-and.html](https://googleprojectzero.blogspot.com/2016/12/chrome-os-exploit-one-byte-overflow-and.html)
5. [https://www.gnu.org/software/libc/manual/html_node/Memory-Allocation-Tunables.html](https://www.gnu.org/software/libc/manual/html_node/Memory-Allocation-Tunables.html)
