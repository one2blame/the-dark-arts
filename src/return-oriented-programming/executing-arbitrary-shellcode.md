# Executing arbitrary shellcode

At this point, we've covered how to use `ROP` to call a `libc` function, how to
chain together multiple `ROP` gadgets to call `libc` functions in succession,
and how to create `ROP` chains that execute `syscall`s directly. What if we
wanted to execute code that wasn't already present in memory?

Remember, the NX/DEP/W^X protection mechanisms were created to combat code
injection techniques that usually executed code directly from the `stack`.
If we debug any normal program today with `gdb` enhancements like `GEF`,
`PEDA`, or `pwndbg`, we can inspect the process's virtual memory mapping with
`vmmap` and see the permissions for each page of memory. Notice that no pages
simultaneously have write and execute permissions. The W^X protection
mechanism tries to delineate between what can be considered "code" and what
can be considered "data", but we see how well that works out when `ROP` is
introduced.

**How do we modify these permissions so that we can execute shellcode?**

We do what the operating system does when it sets permissions for the pages of
our process. If we inspect any program with `strace` we'll find that the
operating system makes multiple calls to the functions `mmap()` and
`mprotect()`. These two functions are used to map files or devices into memory
and to set protections on regions of memory within a process. Now that we fully
understand how to chain successive `libc` function calls, we can create a `ROP`
chain that uses `mprotect()` to modify the permissions of a known location
within memory. [[1]](#references) In order to execute our shellcode, we would
need to follow these steps:

1. Place our shellcode into some known, writeable location within memory.
2. Make a call to `mprotect()` to change the permissions of the page where our
shellcode resides to `r-xp`.
    * `r` stands for `read`
    * `x` stands for `execute`
    * `p` stands for `private`
3. Return to the location in memory containing our shellcode and begin
execution.

Here's an example of how we would do this in `x86-64` using `ROP`. Assume that
the attacker has used an information leak to expose sensitive information
allowing them to discover the base of `libc` in memory:

```python
shellcode = asm(shellcode)          # assemble our shellcode

payload = [
    pop_rdi_pop_rsi_pop_rdx_ret,    # rdi=stdin; rsi=initial, rdx=num_bytes
    0,                              # fd 0 (stdin)
    initial,                        # libc initial data structure
    len(shellcode),                 # length of assembled shellcode
    read_sym,                       # read shellcode into initial from stdin
    pop_rdi_pop_rsi_pop_rdx_ret,    # rdi=initial, rsi=page_size, rdx=perms
    initial,                        # initial now contains our shellcode
    4096,                           # assume page size is 0x1000
    5,                              # PROT_READ|PROT_EXEC
    mprotect_sym,                   # change permissions of initial
    initial                         # ret into our shellcode
]

io.sendline(flat(payload))          # flat() ensures 8 byte alignment
io.send(shellcode)                  # write shellcode into initial with stdin
```

**What's happening here?**

First, we assemble our shellcode - let's assume it's a simple
`sys_execve('/bin/sh', NULL, NULL)` command. Then we construct a `ROP` chain
that will execute `read(STDIN, initial, len(shellcode))`,
`mprotect(initial, 4096, PROT_READ|PROT_EXEC)` [[2]](#references), and then
`ret` into our shellcode in `initial`. Finally, we send the `ROP` chain to the
target and then send the shellcode which will be `read()` into `initial` via
`stdin`.

Once the `read()` operation completes, `mprotect()` will change `initial` from
`rw-p` to `r-xp`, making our shellcode executable. We'll `ret` into `initial`
and begin executing our shellcode.

**How would we do this with mmap?**

1. Create a file containing our shellcode.
2. `open()` the file to acquire a file descriptor.
3. Make a call to `mmap()` using the new file descriptor, mapping the contents
of the shellcode into memory with the permissions `PROT_READ|PROT_EXEC` and the
flags `MAP_PRIVATE|MAP_ANON`. [[3]](#references)
4. Return to the location of the newly mapped memory containing our shellcode
and begin execution.

**How would we do this with mmap without creating a file?**

1. Make a call to `mmap()` with the permissions
`PROT_READ|PROT_WRITE|PROT_EXEC` and the flags `MAP_PRIVATE|MAP_ANON` to create
`rwxp` page(s) in memory. Use other write primitives or gadgets to write
shellcode into the newly mapped `rwxp` memory (via an open socket or `stdin`).
2. Return into the `rwxp` memory containing our shellcode and begin execution.

**What if I don't have gadgets to retrieve the address returned from mmap?**

We use `mmap()`'s `MAP_FIXED` flag in tandem with `MAP_PRIVATE|MAP_ANON` to
control the page-aligned address used by `mmap()`. Then we can hardcode this
known address for use elsewhere in our chain for our eventual jump to our
shellcode.

## What constraints must we consider when attempting to change the permissions of existing memory segments with mprotect?

When using `mprotect()`, the location we provide for the `void *addr` argument
must be aligned to a page boundary. The same goes for `mmap()`, if the
`void *addr` argument provided is not page aligned, an error will occur. The
`void *addr` argument for `mmap()` can be `null` and this is usually the
best case - the kernel will pick a location within memory for us and the
`mmap()` call will return the address in `rax`.

## What are some examples/scenarios in which a program might organically contain RWX buffers?

A simple example would be a program that has NX/DEP protection mechanisms
disabled. We won't come across these types of targets too often, but there are
still some devices in the wild (like IoT devices, routers, etc.) that are
running programs without these exploit mitigation measures. For example, MIPS
Linux didn't support a non-executable stack until around 2015, which can be
considered pretty recent. [[4]](#references)[[5]](#references)

Another example is Just-in-Time (JIT) compilation. JIT compilation can be
summed up as a program creating new executable code that wasn't part of the
original program and then running that code. [[6]](#references) JIT compilation
is supported in most browser applications and sometimes the memory pages that
are created to store and execute the JIT compiled code have `rwx` permissions,
allowing attackers to take advantage of these memory pages to execute arbitrary
code. This type of vulnerability was present in WebKit's JavaScriptCore library
used by the Safari web browser on Apple iOS. [[7]](#references)

[This](https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/) example from
the `oob-v8` challenge in ***CTF 2019** showcases the usage of WebAssembly to
create a `rwx` page within Google Chrome's JavaScript engine, `v8`. The author
demonstrates leaking the address of the `rwx` page using a vulnerability
discovered in `d8`, `v8`'s accompanying JavaScript read-eval-print-loop (REPL)
shell, and then using the same vulnerability to write shellcode into the
discovered memory page to gain code execution.


### References

1. [https://www.ret2rop.com/2018/08/make-stack-executable-again.html](https://www.ret2rop.com/2018/08/make-stack-executable-again.html)
2. [https://man7.org/linux/man-pages/man2/mprotect.2.html](https://man7.org/linux/man-pages/man2/mprotect.2.html)
3. [https://man7.org/linux/man-pages/man2/mmap.2.html](https://man7.org/linux/man-pages/man2/mmap.2.html)
4. [https://lwn.net/Articles/653708/](https://lwn.net/Articles/653708/)
5. [https://lore.kernel.org/patchwork/patch/506101/](https://lore.kernel.org/patchwork/patch/506101/)
6. [https://eli.thegreenplace.net/2013/11/05/how-to-jit-an-introduction](https://eli.thegreenplace.net/2013/11/05/how-to-jit-an-introduction)
7. [https://info.lookout.com/rs/051-ESQ-475/images/pegasus-exploits-technical-details.pdf](https://info.lookout.com/rs/051-ESQ-475/images/pegasus-exploits-technical-details.pdf)
8. [https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/](https://faraz.faith/2019-12-13-starctf-oob-v8-indepth/)
