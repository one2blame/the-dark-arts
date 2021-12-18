# Overcoming ASLR/NX

## What is ROP?

**The below explanation is tailored towards x86 return-oriented programming.
For amd64, the register names would change from E%%->R%% and word size would be
8 bytes instead of 4.**

In a normal program, machine instructions are located within the `.text`
segment of the process's memory. Each instruction is a pattern of bytes to be
interpreted by the processor to change the program's state. The instruction
pointer, `EIP`, governs the sequential flow of a program, pointing to the
instruction that is to be fetched next and advanced by the processor after
each instruction is executed.

A return-oriented program is a particular layout of the `stack` segment within
a process - the `stack` segment being the location within the process's
memory that is pointed to by the `ESP` (more on this later in stack pivoting).
Return-oriented instructions are words on the `stack` that point to an
instruction sequence within the process's memory. In return-oriented
programming, the `stack` pointer, `ESP`, now governs the control flow of the
program and points to the next instruction sequence that will be fetched and
executed.

The execution of return-oriented programs is as follows:
* The word on the `stack` that the `ESP` points to is read and used as the
new value for the `EIP`.
* The `ESP` is incremented by 4 bytes, pointing to the next word on the stack.
* The processor completes execution of the instruction sequence and, if the
provided instruction sequence was a `ROP` gadget, a `ret` would be executed to
repeat this process.

In contrast to normal programs, the `ret` instruction at the end of each
instruction sequence pointed to on the `stack` induces fetch-and-decode
behavior.

## What is JOP?

With the popularity of `ROP`, some work has been done to protect against or
mitigate attacks that leverage `ROP` to deliver malicious payloads and gain
code execution. Proposed defenses include detection of consecutive `ret`
instructions that are suspected `ROP` gadgets [[2]](#references), detection of
`ROP`-inherent behaviors like continuously popping return addresses that always
point to the same memory space [[3]](#references), and the elimination of all
`ret` instructions within a program to remove return-oriented gadgets
[[4]](#references).

Unlike `ROP`, Jump-oriented programming (`JOP`) does not rely upon the `stack`
or `ret` instructions for gadget discovery and gadget chaining. Like `ROP`,
`JOP` finds its gadgets within executable code snippets within the target
binary or in the standard C library, however, `JOP` gadgets end with an
indirect `jmp` instruction. To chain gadgets together, `JOP` specifies two
types of gadgets: the `dispatcher gadget` and `functional gadgets`.

In `ROP`, gadgets are stored on the `stack` and the `ESP` acts as the program
counter for a return-oriented program. In `JOP`, the program counter is any
register that points into the `dispatch table`, and control flow is dictated by
the `dispatcher gadget` that executes the sequence of gadgets within the
`dispatch table`. Each entry within the `dispatch table` points to a
`functional gadget`. In the corpus of gadgets derived from the code contained
within a target binary or shared library, the `dispatcher gadget` is comprised
of jump-oriented gadgets that advance the program counter within the dispatch
table and incorporate the least frequently used registers. This is done to
avoid having the program counter register subjected to side effects of
instructions in `functional gadgets`.

`Functional gadgets` within jump-oriented programming are useful instructions
ending in a sequence that will load the instruction pointer with the result of
a known expression. The primary requirement for `functional gadgets` is that,
by the time the gadget's `jmp %` instruction is executed, the `jmp` must
evaluate to the address of the `dispatcher gadget` or to another gadget that
leads to the `dispatcher`. Sequences that end in a `call` instruction are also
viable candidates for jump-oriented gadgets.

### So how do we gain control flow?

Let's assume we've exploited some vulnerability within a target binary to gain
control of the `EIP`. While `ROP` only requires control over the `EIP` and the
`ESP`, `JOP` requires control over the `EIP` and any memory locations or
registers necessary to run the `dispatcher gadget`. To do this, an
`initializer gadget` is used to fill the relevant registers before jumping to
the `dispatcher gadget`.

### Why would I use JOP over ROP?

As stated earlier, defenses have been proposed to detect `ROP` behavior within
a compromised binary. `JOP`'s lack of reliance on the `stack` and use of `jmp`
and `call` instructions for control flow make it much harder to detect and
create identifiers for. If it is known that your target implements some sort of
protection mechanism to thwart `ROP`, `JOP` is still an option to gain control
over a target and execute code.

## Does a ROP/JOP gadget have to be specifically from the `.text` segment of a binary? What are the characteristics of a usable gadget?

*"[A] gadget is an arrangement of words on the stack, both pointers to
instruction sequences and immediate data words, that when invoked accomplishes
some well-defined task."* [[1]](#references)

The above quote suggests that `ROP` gadgets can be interpreted as a grouping of
words including one or more instruction sequences and immediate values that
encode a logical unit. Gadgets can be built from short instruction sequences
within target binaries, but they also can be derived from libraries used by
target binaries - a commonly used library being the standard C library.

`ROP` gadgets must be constructed so that when the `ret` instruction in the
instruction sequence is executed, `ESP` points to the next gadget to be
executed. The instruction sequences pointed to by gadgets must also reside
within executable segments of memory.

The characteristics of usable `JOP` gadgets are described in the previous
section: ["What is JOP?"](#what-is-jop)

## What kind of primitive(s) and condition(s) might allow you to bypass ASLR via ROP/JOP on a non-PIE binary?

Untrusted Pointer Dereferences, Out-of-bounds Reads, Buffer Over-reads and
similar conditions that provide arbitrary or relative read primitives can be
used in a `ROP`/`JOP` attack to bypass `ASLR`. An attacker would aim to expose
a `libc` address to calculate the base of `libc` within the mapping of the
target's memory.

For a non-`PIE` binary, an attacker could also construct a `ROP` gadget that
would return into the `.plt` section to execute some `libc` function like
`printf` or `puts`. The target of this `printf` or `puts` call would be an
entry within `.got.plt` for a `libc` function that has already been resolved
by the linker. This would expose the address of the target `libc` function to
the attacker, allowing the attacker to calculate the base of `libc` within the
target's memory. This method works on non-`PIE` binaries because the `.plt`
section will be defined at some static address.

## Can the PLT be used to call libc functions even when PIE is enabled? What primitive(s) and condition(s) might be required?

With `PIE` enabled, we must find some way to expose a program address to
calculate the base of the program loaded into the process's memory. This allows
us to calculate the location of the `.plt` section of the program within
memory. The same conditions as in the previous bullet are necessary; Untrusted
Pointer Dereferences, Out-of-bounds Reads, Buffer Over-reads and similar
conditions that provide arbitrary or relative read primitives can be used to
expose sensitive information required to build an exploit.

## How does ROP/JOP evade NX/W^X?

NX, DEP, or the concept of W^X were created in order to combat conventional
code injection techniques that usually executed code directly from the stack.
Attackers found a way around this by using code that was already present within
memory and marked executable, thus inventing return-oriented programming and
jump-oriented programming.

A sufficient set of `ROP`/`JOP` gadgets provides Turing complete functionality
to the attacker, evading the W^X protection mechanism that is designed to
prevent arbitrary code execution.

## References

1. [https://hovav.net/ucsd/dist/rop.pdf](https://hovav.net/ucsd/dist/rop.pdf)
2. [http://www.cs.jhu.edu/~sdoshi/jhuisi650/papers/spimacs/SPIMACS_CD/stc/p49.pdf](http://www.cs.jhu.edu/~sdoshi/jhuisi650/papers/spimacs/SPIMACS_CD/stc/p49.pdf)
3. [https://link.springer.com/chapter/10.1007%2F978-3-642-10772-6_13](https://link.springer.com/chapter/10.1007%2F978-3-642-10772-6_13)
4. [https://www.csc2.ncsu.edu/faculty/xjiang4/pubs/ASIACCS11.pdf](https://www.csc2.ncsu.edu/faculty/xjiang4/pubs/ASIACCS11.pdf)
