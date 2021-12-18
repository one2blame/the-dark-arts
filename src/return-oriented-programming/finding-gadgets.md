# Finding gadgets

## What methods have been used to find ROP/JOP gadgets?

It is known that `ROP` gadgets must end in a `ret` instruction. This allows us
to search through a target binary or shared library to find our useful
instruction sequences that will comprise our gadgets. Luckily for us, in both
the `x86` and `amd64` architectures, instructions are variable-length and
unaligned. This means that we can use both intended and unintended instruction
sequences found within our targets, increasing the collection of available
gadgets.

A primary method presented in Hovav Shacham's paper is to find `ret`
instructions and scan *backwards*, using an algorithm that uses each `ret`
instruction as the root node of a tree and determining if the preceding bytes
represent valid instructions. Each child node of the tree is a valid gadget,
and the algorithm recursively explores the bytes preceding each child to
determine if they are also valid gadgets that can be added to the tree - up to
the maximum length of a valid `x86` instruction.

Hovav contrasts this with finding `ROP` gadgets for `SPARC` targets. We cannot
use unintended instruction sequences to represent `ROP` gadgets because `SPARC`
enforces alignment on instruction read.
[[4]](#references) This essentially reinforces the
point that an attacker must understand the specifics of the particular
architecture they are attacking when condcuting `ROP`.

Finding `JOP` gadgets follows a similar technique. A linear search can be
conducted to find the valid starting bytes for an indirect branch instruction:
`0xff`. This byte is usually followed by a second byte with a specified range
of values. In the same manner as `ROP`, a search is conducted backwards to
discover valid instructions that will comprise the gadget.

In Xuxian Jiang's paper, a series of algorithms are utilized to find gadgets,
deteremine if a gadget is viable for use, and to filter gadgets based upon a
set of heuristics - separating the gadgets into groups that can be used for
the `dispatcher gadget` and `functional gadgets`.
[[5]](#references)

### What constraints on the gadget-searching problem are added or removed by searching for gadgets in RISC architectures (like ARM or MIPS) instead of x86/amd64?

For `x86`/`amd64` architectures, instructions are variable-length and
unaligned. This means that we can consider both intentional and unintentional
instructions for `ROP` gadgets. An unintentional instruction meaning we are
jumping into the *middle* of intentional instructions.

In contrast, `SPARC`, `MIPS`, and `ARM` enforce word alignment for
instructions - this rules out the possibility of deriving our gadgets from
unintentional instructions.

## What are some common tools used to find ROP/JOP gadgets?

The most popular tools used to discover `ROP` gadgets are `ROPgadget` and
`ropper`. The links to these tools can be found in the
[References](#references) section. [[1]](#references), [[2]](#references)
The `pwntools` CTF framework and exploit development library also provides
a `ROP` module that can find `ROP` gadgets and create `ROP` chains.
[[3]](#references)

### How might you implement a similar tool yourself (high-level overview)?

A similar tool that could be implemented quickly would follow some of these
steps:
* User specifies the instruction to search for.
* Dump the target binary or shared library into hex (using something like
`xxd`).
* Use a regular expression to search for the opcode that represents `ret`:
`0xc3`.
* Using the data returned from the results of this search, use another regular
expression to search for the opcode representing the specific instruction
requested by the user.
* Return the line number that contains the gadget we found to the user.

A quick and dirty command line method to find a gadget sequence can be found
in [this blog](https://crypto.stanford.edu/~blynn/rop/) on `ROP`.

## References

1. [https://github.com/JonathanSalwan/ROPgadget](https://github.com/JonathanSalwan/ROPgadget)
2. [https://github.com/sashs/Ropper](https://github.com/sashs/Ropper)
3. [https://docs.pwntools.com/en/stable/rop/rop.html](https://docs.pwntools.com/en/stable/rop/rop.html)
4. [https://hovav.net/ucsd/dist/rop.pdf](https://hovav.net/ucsd/dist/rop.pdf)
5. [https://www.csc2.ncsu.edu/faculty/xjiang4/pubs/ASIACCS11.pdf](https://www.csc2.ncsu.edu/faculty/xjiang4/pubs/ASIACCS11.pdf)
6. [https://crypto.stanford.edu/~blynn/rop/](https://crypto.stanford.edu/~blynn/rop/)
