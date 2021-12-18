# Chaining gadgets to execute code

## Creating a ROP chain on x86

Cool, so now we know how to call one `libc` function, let's call multiple.
Let's assume I have a file I want to read the contents of and the name of the
file is already within memory. I have full control of the `stack` and the
`EIP`/`RIP`. We crafted a `stack` frame to call `system()` with the argument
`'/bin/sh'` in the previous section. How do we craft a `stack` frame that calls
`open()` to open a file descriptor with our target file, `read()` to read the
data within the file into memory, and then `write()` to write the data to
`stdout`?

Again, we assume that the attacker has used an information leak to expose
sensitive information allowing them to discover the base of `libc` in memory.
The attacker also knows the location of the `filename_ptr` in memory.

Here's an example stack frame to accomplish this in `x86`:

```python
p32(open_sym)           # address of open() function within libc
p32(pop_pop_pop_ret)    # pop %; pop %; pop %; ret gadget
p32(filename_ptr)       # arg0: pointer to filename string
p32(0x0)                # arg1: O_RDONLY
p32(0x0)                # arg2: 0 for mode
p32(read_sym)           # address of read() function within libc
p32(pop_pop_pop_ret)    # pop %; pop %; pop %; ret gadget
p32(0x3)                # arg0: assume fd of the target is 3
p32(initial)            # arg1: initial data structure in libc (rw-)
p32(1024)               # arg2: num of bytes to read from fd 3
p32(write_sym)          # address of write() function within libc
p32(pop_pop_pop_ret)    # pop %; pop %; pop %; ret gadget
p32(0x1)                # arg0: fd 1 (stdout)
p32(initial)            # arg1: initial data structure with our file content
p32(1024)               # arg2: num of bytes to write to fd 1 (stdout)
p32(exit_sym)           # address of exit() function within libc
```

**So what's happening in this stack frame?**

When the attacker overwrites the stack frame of the vulnerable function with
the code in this example, the vulnerable function will `ret` into our first
function call: `open()`. `open()` is going to use the `filename_ptr` for its
first argument (`const char *pathname`), `0x0` (`O_RDONLY`) for its second
argument (`int flags`), and `0x0` for its third argument (`mode_t mode`).

If the `open()` function is successful, `eax` will contain the file descriptor
of the now open file pointed to by `filename_ptr` - in this example we're
assuming the file descriptor is `3`. Next, we use a `pop %; pop %; pop %; ret`
gadget to `pop` our arguments to `open()` off of the stack. The operand for
these `pop %` instructions doesn't really matter, so long as it's not `ESP`.

The `pop %; pop %; pop %; ret` gadget will now `ret` into our `read()` address
on the `stack`, `pop`ing the word off of the `stack` and executing `libc`'s
`read()` function. `read()` is going to use `0x3` for its first argument
(`int fd`), `initial` for its second argument (`void* buf`), and `1024` for
its third argument (`size_t count`). If the `read()` function is successful,
`eax` will contain the number of bytes read from the file. We `ret` into our
gadget to `pop` all of `read()`'s arguments off the `stack`, then we `ret`
into our `write()` call.

`write()` is going to use `0x1` (`stdout`) for its first argument (`int fd`),
`initial` for its second argument (`const void *buf`), and `1024` for its third
argument (`size_t count`). If the `write()` function is successful, `eax` will
contain the number of bytes that was written to `stdout`. We `ret` into our
gadget and `pop` all of `write`'s arguments off the `stack`, then we `ret` into
our `exit()` call.

### Can you explain in greater detail why `pop %; pop %; ret`-esque gadgets are needed for building ROP chains on x86?

If we didn't use a `pop %; pop %; pop %; ret` gadget to clear our function
arguments off of the `stack`, the `stack` frame would look something like this:

```python
p32(open_sym)
p32(read_sym)
p32(filename_ptr)
p32(0x0)
p32(0x0)
```

**So what's the problem here?**

When `open()` finishes, we'll `ret` into our `read()` call. This isn't going
to work out well for us though because `read()`'s arguments don't make any
sense now. Our first argument for `read()` points to a `null` byte, which isn't
a `filename_ptr`.

It's necessary for us to use `pop %; pop %; pop %; ret` gadgets in order to
clear the previous function call's arguments from the stack. These gadgets
allow us to effectively chain multiple function calls. [[1]](#references)

## Creating a ROP chain on x86-64

Let's do the same `open()`, `read()`, `write()`, `exit()` `ROP` chain on
`x86-64`:

```python
p64(pop_rdi_ret)            # load filename_ptr into rdi
p64(filename_ptr)           # arg0: pointer to filename string
p64(pop_rsi_pop_r15_ret)    # load flags into rsi
p64(0x0)                    # arg1: O_RDONLY
p64(0xcafebabe)             # dummy bytes loaded into r15
p64(pop_rdx_ret)            # load mode into rdx
p64(0x0)                    # arg2: 0 for mode
p64(open_sym)               # address of open() function within libc
p64(pop_rdi_ret)            # load fd into rdi
p64(0x3)                    # arg0: assume fd of the target is 3
p64(pop_rsi_pop_r15_ret)    # load initial into rsi
p64(initial)                # arg1: initial data structure in libc (rw-)
p64(0xcafebabe)             # dummy bytes loaded into r15
p64(pop_rdx_ret)            # load num of bytes to read into rdx
p64(1024)                   # arg2: num of bytes to read from fd 3
p64(read_sym)               # address of read() function within libc
p64(pop_rdi_ret)            # load fd 1 (stdout) into rdi
p64(0x1)                    # arg0: fd 1(stdout)
p64(pop_rsi_pop_r15_ret)    # load initial into rsi
p64(initial)                # arg1: initial data structure with file content
p64(0xcafebabe)             # dummy bytes loaded into r15
p64(pop_rdx_ret)            # load num of bytes to write into rdx
p64(1024)                   # arg2: num of bytes to write to fd 1 (stdout)
p64(write_sym)              # address of write() function within libc
p64(exit_sym)               # address of exit() function within libc
```

**So what's happening in this stack frame?**

The operations that take place within this `stack` frame are exactly the same
as what took place in the `x86` example, we're just using `ROP` gadgets to
ensure that we're following the `x86-64` calling convention. Each `pop %; ret`
gadget is `pop`ing our arguments from the `stack` into the correct registers
for each `libc` function call. You can see that not everything is perfect,
though. The `pop rsi; pop r15; ret` gadget is `pop`ing a dummy value from the
`stack` into `r15`. We're not always going to find the perfect gadget to
`pop %` just one value into our target register - we have to make due with
what's available.

## Creating a ROP chain to execute syscalls on x86 and x86-64

Alright, so now that we understand how to chain gadgets to make multiple `libc`
calls, here are some examples of `ROP` chains that execute `syscall`s in `x86`
and `x86-64`.

Here's an example `ROP` chain that executes
`sys_open(filename_ptr, O_RDONLY, 0)` in `x86`:

```python
p32(pop_ebx_ret)    # load filename_ptr into ebx
p32(filename_ptr)   # arg0: pointer to filename string
p32(pop_ecx_ret)    # load flags into ecx
p32(0x0)            # arg1: O_RDONLY 
p32(pop_edx_ret)    # load mode into edx
p32(0x0)            # arg2: 0 for mode
p32(pop_eax_ret)    # load syscall number into eax
p32(0x5)            # syscall number for sys_open
p32(int_80_ret)     # int 0x80; ret gadget
```

Here's an example `ROP` chain that executes
`sys_open(filename_ptr, O_RDONLY, 0)` in `x86-64`:

```python
p64(pop_rdi_ret)    # load filename_ptr into rdi
p64(filename_ptr)   # arg0: pointer to filename string
p64(pop_rsi_ret)    # load flags into rsi
p64(0x0)            # arg1: O_RDONLY 
p64(pop_rdx_ret)    # load mode into rdx
p32(0x0)            # arg2: 0 for mode
p32(pop_rax_ret)    # load syscall number into rax
p64(0x2)            # syscall number for sys_open
p64(syscall)        # syscall; ret
```

As you've probably noticed these two `ROP` chains seem super simple and super
similiar, but it won't always be that way. As I said earlier, creating a `ROP`
chain is a test of creativity, and you won't always be able to load your
registers with simple `pop %; ret` gadgets.

### How might one combine multiple gadgets into one register-populating pseudo-gadget? What are some useful non-pop instructions for doing this in amd64?

You can definitely find gadgets that will `pop` all of your necessary arguments
into their respective registers. If we found something like:
`pop rdi; pop rsi; pop rdx; ret` in code, we could use this gadget before
making our `read()`, `write()`, and `open()` calls. [[2]](#references)

In `x86`, there's an interesting instruction called `popa`/`popad` that `pop`s
data into all general-purpose registers. This is definitely useful if you need
to intialize all of your registers prior to executing further code.
[[3]](#references)

**So what if I can't find a pop instruction for my target register?**

If you can't find a `pop` instruction to directly load a register, you still
might have some options if you use `xor` and `xchg`.

The concept in using `xor` to load a register is as follows [[2]](#references):

1. `xor` your target register against itself to zero it out.
2. `pop` the data you want to load into a different register.
3. `xor` your target register against the other register, duplicating the data
into your target register.

The concept in using `xchg` to load a register is as follows
[[2]](#references):

1. `pop` the data you want to load into a different register.
2. execute an `xchg` with your target register and the register containing your
data.

**Using the `xchg` method will swap the data between the two registers. Ensure
you reload the register you used to execute `xchg` if you intend to use it
later.**

The `mov %, %; ret` gadget is also a viable method to load target registers.
Generating `ROP` chains to create exploits is just a test of the exploiter's
creativity.

## Under what conditions would one need to stack pivot? What are some locations an attacker could store a second-stage chain for use in a stack pivot?

These `ROP` chains can get pretty long, sometimes we might not be able to write
more than a handful of words to the `stack`. In these conditions, we'll have to
conduct a `stack` pivot.

A `stack` pivot is a technique that relocates the `stack` to some other
location in memory. This allows us to completely control the contents of the
`stack`, and this is where we can place more words for our `ROP` chain. When
we pivot the `stack` to our new location, we can continue executing our `ROP`
chain without any of the pesky restrictions we faced on the original `stack`.

In order to pivot the `stack`, we `pop` the location containing the rest of
our `ROP` chain into the `ESP`/`RSP` register. Then, when our `pop rsp; ret`
gadget executes `ret`, the `stack` will be pointing to the location in memory
containing our second stage `ROP` chain.

An attacker could place a second stage `ROP` chain in any know read/writeable
location in memory. These locations could include the `heap`, the `stack`, a
buffer in `.bss` or `.data`, or (my personal favorite) `libc`'s `initial` data
structure. [[2]](#references)

### I'm not familiar with the initial libc structure, can you explain what it is?

`initial` is a data section within `libc` that is read/writeable. If you know
the base of `libc` within memory and you know the version of `libc` being used
by a target program, you can determine the location of `libc`'s `initial` data
structure within memory. It's a useful location to store file data that's been
read into memory, or second-stage `ROP` chains because the data strucuture is
usually empty.

The `initial` data structure is used by the `libc` function `atexit()` to store
a list of function pointers that will be called when `exit()` is called. In the
`Left` challenge from **0x00 CTF 2017**, the `initial` data structure plays a
role in gaining control of the `RIP`.

Function pointers within `initial` are `xor`d with some secret in thread-local
storage (TLS) and then rotated left 17 bits. In the `Left` challenge the
attacker needs to expose an entry within `initial`, derive the secret from the
entry, and then replace the entry in `initial` with a `one_gadget`. Then, when
`exit()` is called, the `one_gadget` is executed. This technique was
necessary because the target program had `Full RELRO` protections enabled,
preventing the attacker from overwriting a `.got.plt` entry.
[[4]](#references)

All that being said, yes if you know the location of `.data` or `.bss` within
the program it's just as easy to write your second-stage `ROP` chain there.
However, if the program has the `PIE` protection mechanism enabled and you
don't feel like deriving the base of the program within memory, `initial` is
another read/writeable location that can be easily derived from the base of
`libc`. Use this location at your own discretion though, especially if some
information contained with `initial` seems important.

### References

1. [https://hovav.net/ucsd/dist/blackhat08.pdf](https://hovav.net/ucsd/dist/blackhat08.pdf)
2. [https://trustfoundry.net/basic-rop-techniques-and-tricks/](https://trustfoundry.net/basic-rop-techniques-and-tricks/)
3. [http://qcd.phys.cmu.edu/QCDcluster/intel/vtune/reference/vc246.htm](http://qcd.phys.cmu.edu/QCDcluster/intel/vtune/reference/vc246.htm)
4. [https://github.com/SPRITZ-Research-Group/ctf-writeups/tree/master/0x00ctf-2017/pwn/left-250](https://github.com/SPRITZ-Research-Group/ctf-writeups/tree/master/0x00ctf-2017/pwn/left-250)
