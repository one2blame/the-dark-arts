# House of Orange

The original **House of Orange** technique used a heap overflow vulnerability
to target the `top chunk` in its first stage of exploitation. Due to this fact,
we can determine that the **House of Orange** fits within this section of heap
exploitation.

## The technique

The **House of Orange** is an interesting, but somewhat convoluted, technique
to gain arbitrary code execution of a vulnerable process. The
**House of Orange** is executed in three stages:

* Leverage a heap overflow vulnerability to overwrite the size field of the
`top chunk`.
  * Overwrite the `top chunk` with a small size, fooling `malloc()` in future
requests to believe that the `top chunk` is smaller than it actually is.
  * The new `top chunk` size must be page-aligned and the `prev_inuse` bit must
be set in order to pass `malloc()` checks.
  * The attacker forces the program to make another `malloc()` call with a size
larger than what is currently written to the `top chunk` size field.
  * This `malloc()` call will cause `malloc()` to `mmap()` a new segment of
heap memory. `malloc()` will also determine that the new segment of heap memory
and the `top chunk` are not contiguous, causing `malloc()` to `free()` the
remaining space of the `top chunk`.
  * This newly free chunk will be too large for the fastbin, it will be linked
into the unsortedbin.

* Use the same heap overflow vulnerability and chunk to overwrite the newly
freed `top chunk` that resides in the unsortedbin.
  * The attacker forges the metadata for a fake chunk, setting the chunk size
to `0x61`, and setting the `bk` pointer to a chunk that overlaps `_IO_list_all`
in `glibc`
.
  * The attacker will use this to conduct an **Unsortedbin Attack**, writing
the memory address of the unsortedbin head in the `main arena` to
`_IO_list_all`.
  * The attacker uses the heap overflow to write a fake `_IO_FILE` struct into
the heap, forging a `vtable_ptr` that points back into attacker controlled
memory with the intent of overwriting the `overflow` method of the struct to
`system()`.

* The attacker requests a chunk smaller than the forged chunk that was just
created, causing `malloc()` to sort the free chunk into the smallbin,
triggering an **Unsortedbin Attack**.
  * `malloc()` attempts to follow our forged `bk`, however, chunk metadata
checks will cause `malloc()` to call `__malloc_printerr()`, leading to a
`SIGABRT`.
  * When the program begins its exit procedures, it attempts to clear the
buffers of all open file streams, including our forged one.
  * `glibc` follows `_IO_list_all` which now points to the `main_arena`. The
`main_arena` fails `_IO_FILE` struct checks, and `glibc` moves on to the next
`_IO_FILE` struct pointed to by the `main_arena`'s fake `chain` member - our
smallbin.
  * `glibc` inspects the forged `_IO_FILE` struct the attacker created in the
heap using a heap overflow and executes the `overflow` method listed in the
`vtable` of the forged `_IO_FILE` struct. The attacker has overwritten the
`overflow` method listed in the `vtable` to point to `system()`.
  * The address of the `_IO_FILE` is passed to this call to `system()` - the
attacker ensures that the string `/bin/sh\0` resides at this location, the
very first word of bytes in the forged `_IO_FILE` struct.

### Some notes

Like I said, this technique is convoluted. The attacker needs to have the
following in order to exploit this vulnerability:

* Ability to edit chunk data
* Ability to control `malloc()` allocation size
* Heap and `glibc` address leak if `ASLR` is enabled
* Heap overflow

## Patch

There doesn't seem to be any specific patch that attempts to mitigate
exploitation using the **House of Orange**. Because there are so many
conditions necessary to effectively exploit this technique, the summation of
the mitigations applied to `glibc` over the years have made this technique
obsolete.

For instance, the patch applied to actually check the validity of the `bk`
pointer in the unsortedbin causes the **Unsortedbin Attack** used to execute
this technique to fail. In addition, `glibc` after version `2.28` no longer
traverses through all open file streams at program exit to call the `overflow`
method of the stream. This mitigation thwarts this techniques use of
**File Stream Oriented Programming** to gain code execution.

## References

1. [https://4ngelboy.blogspot.com/2016/10/hitcon-ctf-qual-2016-house-of-orange.html](https://4ngelboy.blogspot.com/2016/10/hitcon-ctf-qual-2016-house-of-orange.html)
2. [https://ctf-wiki.github.io/ctf-wiki/pwn/linux/io_file/fsop/](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/io_file/fsop/)
