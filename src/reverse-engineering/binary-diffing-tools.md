# Binary diffing tools

## Diaphora

[Diaphora](#references), Greek for "difference", is a binary diffing project
and tool started by [Joxean Koret](http://www.joxeankoret.com/), originally
released in 2015. Diaphora is advertised as the most advanced binary diffing
tool that works as an IDA Pro plugin. [IDA Pro](#references) is a licensed
disassembler and debugger for state-of-the-art binary code analysis.

Diaphora features all of the common binary diffing techniques such as:

* Diffing assembler.
* Diffing control flow graphs.
* Porting symbol names and comments.
* Adding manual matches.
* Similarity ratio calculation.
* Batch automation.
* Call graph matching calculation.

And it also comes with many more advanced features listed in its `README.md`.
An IDA Pro license comes at a hefty price, however, the contributors to
Diaphora have advertised that support for Ghidra and Binary Ninja are being
actively developed. [Ghidra](#references) is a free software reverse
engineering suite of tools developed by the National Security Agency (NSA).
[Binary Ninja](#references) is another licensed reversing platform developed by
Vector 35.

## Bindiff

[Bindiff](#references) is another binary diffing tool developed by Zynamics and
works as a plugin for IDA Pro, as well. Here are some of its use cases as
advertised on their web page:

* Compare binary files for x86, MIPS, ARM, PowerPC, and other architectures
supported by IDA Pro.
* Identify identical and similar functions in different binaries.
* Port function names, anterior and posterior comment lines, standard comments
and local names from one disassembly to the other.
* Detect and highlight changes between two variants of the same function.

It's not all about IDA Pro; As of March 1, 2020, Zynamics released Bindiff 6
which provides experimental support for Ghidra. Open-source research has been
conducted to generate a plugin for these new features and a plugin called
[BinDiffHelper](#references) is currently available that aims to provide
easy-to-use support for Bindiff 6 with Ghidra.

## Radiff2

[Radiff2](#references) is Radare2's binary diffing utility.
[Radare2](#references) is a rewrite from scratch of Radare in order to provide
a set of libraries and tools to work with binary files. The Radare project
started as a forensics tool, a scriptable command-line hexadecimal editor able
to open disk files, but later added support for analyzing binaries,
disassembling code, debugging programs, attaching to remote gdb servers, and
so on.

Radiff2 is open-source and attempts to provide the same utility and
functionality as the binary diffing tools listed above without having to cater
to licensed tools like IDA Pro and Binary Ninja.

## References

1. [https://github.com/joxeankoret/diaphora](https://github.com/joxeankoret/diaphora)
2. [https://www.hex-rays.com/products/ida/](https://www.hex-rays.com/products/ida/)
3. [https://ghidra-sre.org/](https://ghidra-sre.org/)
4. [https://binary.ninja/](https://binary.ninja/)
5. [https://www.zynamics.com/bindiff.html](https://www.zynamics.com/bindiff.html)
6. [https://github.com/ubfx/BinDiffHelper](https://github.com/ubfx/BinDiffHelper)
7. [https://r2wiki.readthedocs.io/en/latest/tools/radiff2/](https://r2wiki.readthedocs.io/en/latest/tools/radiff2/)
8. [https://github.com/radareorg/radare2](https://github.com/radareorg/radare2)
