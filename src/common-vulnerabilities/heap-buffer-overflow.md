# Heap buffer overflow

## What is a heap?

**Heaps** are contiguous blocks of memory chunks which `malloc()` allocates
to a process. Heaps are dynamic in nature, so memory can also be `free()`d
by a process when the memory is no longer needed. Heap memory is global and
can be accessed and modified from anywhere within the process when referenced
with a valid pointer. Heaps are treated differently depending on whether
they belong to the main arena or not - more on arenas later.

**Heaps** can be created, extended, trimmed, and destroyed. The main arena
heap is created during the first request for dynamic memory while heaps for
other arenas are created with the `new_heap()` function. The main arena
heap grows and shrinks with the use of the `brk()` or `sbrk()` system calls.
These system calls are used to change the location of the program break,
defining the end of the process's `data` segment. Increasing the
program break allocates memory to the process; decreasing the break
deallocates memory. [[1]](#references)

### What is a chunk?

Chunks are the fundamental unit of memory that `malloc()` deals in, they are
pieces of heap memory. An allocated chunk in the heap is structured like this:

```
            word/qword          |           word/qword

+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           prev size           |           chunk size        |A|M|P|
|   (not used while allocated)  |                                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           user data           |           user data               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           user data           |       size of next chunk    |A|M|P|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

The flags at the end of the second `word`/`qword` represent these chunk
properties:

* **A** (`NON_MAIN_ARENA`) - `0` for chunks in the main arena. Each thread
spawned receives its own arena and for those chunks, this bit is set.
* **M** (`IS_MMAPPED`) - The chunk was obtained using `mmap()`.
* **P** (`PREV_INUSE`) - `0` when the previous chunk is free. The first chunk
allocated will always have this bit set. [[3]](#references)

The minimum usable chunk size is `0x20`, and chunk sizes increase in
increments of 16 bytes: `0x20`, `0x30`, `0x40`, etc.

#### The top chunk

The top chunk is the "topmost available chunk, i.e. the one bordering the end
of available memory". Requests are only serviced from a top chunk when they
can't be serviced from any other bins in the same arena. `malloc()` keeps
track of the remaining memory in a top chunk using its `chunk_size` field,
and the `PREV_INUSE` bit of a top chunk is always set. A top chunk always
contains enough memory to allocate a minimum-sized chunk and always ends on a
page boundary. [[4]](#references)

### What is an arena?

`malloc()` administrates a process's heaps using `malloc_state` structs, or
**arenas**. **Arenas** contain **bins**, data strucutures used for recycling
free chunks of heap memory. Definitions for each type of **bin** can be found
below:

#### fastbins

* A collection of singly linked, non-circular lists that each hold free chunks
of a specific size.
* The first `word`/`qword` of a chunk's user data is used as the forward
pointer (`fd`) when linked into a fastbin.
* fastbins are LIFO data structures.
* Free chunks are linked into the fastbin if the tcachebin is full.
* For memory requests, the fastbin is searched after the tcachebin and before
any other bin.
* There are 10 fastbins with chunk sizes: `16..88` bytes. [[2]](#references)

#### unsortedbin

* A doubly linked, circular list that holds free chunks of any size;
essentially used to optimize resource requests.
* Free chunks are linked directly into the head of an unsortedbin when the
tcachebin is full or they are outside tcachebin size range.
* The first `word`/`qword` of a chunk's user data is used as the forward
pointer (`fd`) and the second `word`/`qword` is used as a backwards pointer
(`bk`) when linked into an unsortedbin.
  * In versions of GLIBC compiled without the tcachebin, free chunks are linked
directly into the head of an unsortedbin when they are outside fastbin range.
* The unsortedbin is searched after the tcachebin, fastbin, and smallbins when
the request size fits those ranges, but before the largebins.
* unsortedbin searches begin from the back and move towards the front.
* If a request fits a chunk within the unsortedbin, the search is stopped and
the memory is allocated, otherwise the chunk is sorted into a smallbin or
largebin. [[2]](#references)

#### smallbins

* A collection of doubly linked, circular lists that each hold free chunks of a
specific size.
* Free chunks are only linked into a smallbin when the chunk's arena sorts the
unsortedbin.
* The first `word`/`qword` of a chunk's user data is used as the forward
pointer (`fd`) and the second `word`/`qword` is used as a backwards pointer
(`bk`) when linked into a smallbin.
* smallbins are FIFO data structures.
* For memory requests, smallbins are searched after the tcachebin, after
fastbins if the request was in fastbin size range, and before any other bins
are searched if the request was in smallbin range.
* There are 62 smallbins with chunk sizes: `16..512` bytes (32-bit),
`16..1024` bytes (64-bit). [[2]](#references)

#### largebins

* A collection of doubly linked, circular lists that each hold free chunks
within a range of sizes.
* Free chunks are only linked into a largebin when the chunk's arena sorts the
unsortedbin.
* largebins are maintained in descending size order.
  * The largest chunk is accessible via the bin's `fd` pointer.
  * The smallest chunk is accessible via the bin's `bk` pointer.
* The first `word`/`qword` of a chunk's user data is used as the forward
pointer (`fd`) and the second `word`/`qword` is used as a backwards pointer
(`bk`) when linked into a largebin.
* The first chunk of its size linked into a largebin has the third and fourth
`word`/`qword`s repurposed as `fd_nextsize` and `bk_nextsize`, respectively.
* The nextsize pointers are used to form another doubly linked, circular list
holding the first chunk of each size linked into that largebin. Subsequent
chunks of the same size are added after the first chunk of that size.
* The nextsize pointers are used as a skip list.
* For memory requests, largebins are searched after an unsortedbin, but before
a binmap search.
* There are 63 largebins with chunk sizes starting at: `512` bytes (32-bit),
`1024` bytes (64-bit). [[2]](#references)

#### What is a tcache?

A tcache behaves like an arena but, unlike arenas, is not shared between
threads of a process. They are created by allocating space on a heap belonging
to their thread's arena and are freed when the thread exits. The purpose of a
tcache is to relieve thread contention for malloc's resources by allocating to
each thread its own collection of chunks.

* The tcache is present in GLIBC versions >= 2.26.
* A tcache takes the form of a `tcache_perthread` struct which holds the heads
of 64 tcachebins preceded by an array of counters which record the number of
chunks in each tcachebin.
* tcachebins are singly linked, non-circular lists of free chunks of a specific
size.
* When a tcachebin reaches it's maximum number of chunks, free chunks of that
bin's size are instead treated as they would be without a tcache present.
* There are 64 tcachebins with chunk sizes: `12..516` bytes (32-bit),
`24..1032` bytes (64-bit). [[2]](#references)

## Heap overflow

OK, so all of the previous information was great right? It may seem like a lot,
but understanding the intricacies of the GLIBC Heap is important to effectively
exploit it. Now, I'll finally define what a heap overflow is:

"A heap overflow is a buffer overflow, where the buffer that can be overwritten
is allocated in the heap portion of memory, generally meaning that the buffer
was allocated using a routine such as malloc()." [[5]](#references)

Here's an example of code that presents a heap overflow vulnerability:

```c
#define BUFSIZE 256
int main(int argc, char **argv) {
    char *buf;
    buf = (char *)malloc(sizeof(char)*BUFSIZE);
    strcpy(buf, argv[1]);
}
```

No bounds checking is conducted on `argv[1]` before its data is copied into
`buf`. `strcpy()` could possibly copy data from `argv[1]` well past the bounds
of the chunk that was allocated on the heap to hold `buf`.

Heap overflows are a dangerous weakness for a an application to have and, given
the right circumstances, attackers can utilize a variety of techniques to gain
arbitrary code execution. Techniques to exploit heap overflows have been
studied since the early 2000's and we'll cover a number of these techniques in
the following sections.

## References

1. [https://www.blackhat.com/presentations/bh-usa-07/Ferguson/Whitepaper/bh-usa-07-ferguson-WP.pdf](https://www.blackhat.com/presentations/bh-usa-07/Ferguson/Whitepaper/bh-usa-07-ferguson-WP.pdf)
2. [https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/](https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/)
3. [https://heap-exploitation.dhavalkapil.com/diving_into_glibc_heap/malloc_chunk.html](https://heap-exploitation.dhavalkapil.com/diving_into_glibc_heap/malloc_chunk.html)
4. [https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/](https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/)
5. [https://cwe.mitre.org/data/definitions/122.html](https://cwe.mitre.org/data/definitions/122.html)
