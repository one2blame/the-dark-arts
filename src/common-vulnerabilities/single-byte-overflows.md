# Single Byte Overflows

Single byte overflows are pretty dangerous bugs to have in a program, given the
right conditions. A stack-based, single byte, `NULL` buffer overflow was
demonstrated in this [this post](https://seclists.org/bugtraq/1998/Oct/109) in
1998, used to overwrite the LSB of the `ebp` causing the stack frame to be
relocated to a lower address in memory - a location in the stack that the user
controls and can forge a fake stack frame.

In 2014, Google Project Zero releases *The poisoned NUL byte, 2014 edition*,
demonstrating that a heap-based, single byte, `NULL` buffer overflow can also
be used to gain code execution. [[2]](#references)

## Techniques

The following subsections cover a white paper from Accenture Security that goes
in-depth on different methods of exploiting a one byte heap-based buffer
overflow to gain overlapping chunks and possibly code execution.
[[3]](#references)

### Extending free chunks

A diagram for this technique can be found in section 3.2.1 of
[[3]](#references). With a one byte heap-based buffer overflow, the attacker
will write actual information to the `size` field of a free chunk, increasing
its size. An allocation larger than the corrupted chunk's original size will
cause the chunk to overlap into succeeding chunks.

This scenario relies on the fact that `malloc()` does not check the `prev_size`
field of the succeeding chunk when allocating a previously `free()`d chunk.

### Extending allocated chunks

This technique is similar to the one above, it's just that the series of
operations is reordered. A diagram for this technique can be found in section
3.2.2 of [[3]](#references). Essentially, the corruption happens before the
victim, corrupted chunk is `free()`d. An attacker writes one byte, increasing
the size of the corrupted chunk. After the corrupted chunk is `free()`d,
another allocation is requested with a size greater than the original size
of the corrupted chunk, causing the corrupted chunk's user data to overlap
the succeeding chunk.

This technique exploits the fact that `free()` has no ability to determine if
the corrupted chunk's `size` field is supposed to be larger or smaller, as the
only location that contains the chunk's size metadata is the `size` field
corrupted by the one byte heap-based buffer overflow.

### Shrinking free chunks

Here's our heap-based, single byte, `NULL` buffer overflow - a memory
corruption vulnerability that can lead to some interesting outcomes. A good
demonstration of this technique exists on the `how2heap` repo,
[here](#references). The diagram for this technique can be found in section
3.2.3 of [[3]](#references).

The initial state of the program involves having three chunks allocated on the
heap, all too large for the fastbin. The chunk in the middle of these three
will have a size such that a `NULL` byte buffer overflow from the preceding
chunk will overwrite the `size` field of the middle chunk, causing it to shrink
in size. The example provided sets the middle chunk's size to `0x210` and,
after the `NULL` byte overflow, its size is set to `0x200`.

Before the memory corruption occurs, the middle chunk is `free()`d. This is
required because the `prev_size` field of the succeeding chunk must be set to
`0x210`. The attacker conducts the `NULL` byte buffer overflow and sets the
free chunk's size to `0x200`. **In later updates to `glibc`, the `prev_size`
field and the `size` field of the chunk about to be backwards consolidated is
now checked for consistency. Attackers must now write a valid `prev_size` field
to the succeeding chunk before attempting this backwards consolidation.**

Two chunk are now allocated from this newly `free()`d space, one chunk that's
not within fastbin range, and subsequent chunks that are. We want chunks that
are in fastbin range to avoid having these chunks subjected to `malloc()`'s
consolidation and sorting behavior. We free the first chunk of the two chunks
we just allocted, placing it in the unsortedbin.

Almost there, the attacker frees the third chunk of the original three, causing
`free()` to inspect the `prev_size` field of the third chunk and consolidating
it with the chunk that we just `free()`d into the unsortedbin. This free space
now overlaps the fastbin sized chunk we allocated earlier.

Finally, we allocate a chunk large enough to overlap the fastbin sized chunk
that still remains. Because of this heap-based, one byte, `NULL` buffer
overflow, we have allocated two chunks that overlap one another which can
easily lead to the implementation of other exploitation primitives.

### Exploitation

In [[3]](#references), further dicussion is provided on how to use overlapping
chunks to leak sensitive information and gain code exeuction. In section 4.3.2,
the authors demonstrate that they can create overlapping unsortedbin size
chunks.

They `free()` the unsortedbin eligible chunk, and the pointer to the head of
the unsortedbin, which resides in the `main_arena`, is written to the `fd` and
`bk` pointers of the chunk. Because they still maintain a chunk overlapping
this now free chunk, they can leak the memory of the free chunk to obtain its
`fd` and `bk` pointers, providing them a `glibc` leak.

Not covered in this white paper, but possible nonetheless, is that the
allocation and `free()`ing of two fastbin eligible chunks can lead to a heap
location leak. If these two fastbin eligible chunks are `free()`d while being
overlapped by a much larger, allocated chunk, the attacker could feasibly read
the `fd` pointers of these fastbin free chunks to derive the base of the heap.

Finally, to gain code execution, the authors of this white paper use a Global
Offset Table (GOT) overwrite. For the application they are exploiting, the
program maintains sensitive structures within the heap and, within these
structures, there are pointers that will be used to write incoming data from
a socket connection. The authors generate two of these structures in the heap,
one to leak `glibc` and one to conduct arbitrary writes. The authors use the
`glibc` leak to derive the location of the GOT - for this application's
environment, the GOT is always a fixed offset away from the `glibc` base. Once
they've derived the location of `free@GOT`, they corrupt a heap structure's
write pointer to point to `free@GOT`. After this, they send a message which
causes the program to overwrite `free@GOT` with the pointer to `system@libc`.
Coercing the program to call `free@plt` leads to `system@libc` with a command
they provide.

## Patch

A patch that thwarts the shrinking of chunks to gain overlapping chunks was
implemented on AUG 2018. This is referenced by the note earlier, as this patch
conducts a consistency check between the `next->prev_size` and `victim->size`
of a `victim` within the unsortedbin before sorting or consolidating the
`victim`. [[5]](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=b90ddd08f6dd688e651df9ee89ca3a69ff88cd0c)

## References

1. [https://seclists.org/bugtraq/1998/Oct/109](https://seclists.org/bugtraq/1998/Oct/109)
2. [https://googleprojectzero.blogspot.com/2014/08/the-poisoned-nul-byte-2014-edition.html](https://googleprojectzero.blogspot.com/2014/08/the-poisoned-nul-byte-2014-edition.html)
3. [https://www.contextis.com/en/resources/white-papers/glibc-adventures-the-forgotten-chunks](https://www.contextis.com/en/resources/white-papers/glibc-adventures-the-forgotten-chunks)
4. [https://github.com/shellphish/how2heap/blob/master/glibc_2.23/poison_null_byte.c](https://github.com/shellphish/how2heap/blob/master/glibc_2.23/poison_null_byte.c)
5. [https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=b90ddd08f6dd688e651df9ee89ca3a69ff88cd0c](https://sourceware.org/git/?p=glibc.git;a=commitdiff;h=b90ddd08f6dd688e651df9ee89ca3a69ff88cd0c)
