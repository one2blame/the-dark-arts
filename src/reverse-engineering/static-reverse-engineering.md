# Static reverse engineering

Reverse engineers use static reverse engineering to inspect programs at rest,
meaning not loaded into memory or currently executing. Reverse engineers
utilize a series of tools to statically inspect a program in order to derive
its purpose and functionality. Starting at the shallow end of analysis, the
following tools are useful for static reverse engineering:

* `strings` - the strings utility outputs all printable character sequences
that are at least 4 characters long for a given file. This tool is useful for
reverse engineers as it provides an initial peek at what functionality might
exist within the target program.
* `file` - the file utility gives a best-effort determination of what type of
file the reverse engineer is inspecting. `file` looks for common ASCII strings,
but also checks a file to see if it contains data structured according to known
file formats like ELFs.
* `PEiD` - a Windows tool that's useful for identifying the compiler used to
build a Windows PE and any tools used to obfuscate the PE being inspected.

Some tools that provide a little more sophistication for static reverse
engineering are as follows:

* `nm` - this utility lists the symbols in a provided object file or executable
. Using `nm` on object files just shows an object file's exported global
varibles and symbols - an executable provides similar output, however, external
functions that the program links against will also be present.
* `ldd` - **list dynamic dependencies**, this will list the libraries a
dynamically linked executable uses / expects in order to implement its
functionality.
* `objdump` - an extremely versatile utility for ELF inspection, this program
can provide a reverse engineer the following information about a program:
  * Program section headers - a summary of the information for each section in
the program - e.g.: `.text`, `.rodata`, `.dynamic`, `.got`, `.got.plt`, etc.
  * Private headers - program memory layout information and other information
required by the loader, e.g. a list of required libraries - produces similar
output to `ldd`.
  * Debugging information
  * Symbol information
  * Disassembly listing - `objdump` performs a **linear sweep disassembly** of
the sections of the file marked as code. `objdump` outputs a text file of this
listing, commonly known as a `dead listing`.
* `c++filt` - a tool that understands and deciphers the symbol name mangling
scheme of different compilers for C++ applications. Compilers mangle symbol
names with some standardized formatting in order to avoid symbol collisions
of symbols within different namespaces. `c++filt` not only de-mangles the
symbol output of tools like `nm` on `g++` compiled programs, but is also
able to identify and fix paramters passed to previously mangled symbols.

Finally, at the deep end of static reverse engineering tools:

* `ndisasm` - a **stream disassembler**, this utility doesn't require the data
inspected to be a valid program. It just takes a stream of bytes and attempts
to accurately disassemble the bytes to provide readable assembly code. Of
course you'll have to provide `ndisasm` with the suspected target architecture
and pointer size (32bit or 64bit).
* `IDAPro` - created by HexRays, IDAPro is a static reverse engineering tool
that conducts disassembly and decompliation of a target program and uses a
proprietary technology called FLIRT (Fast Library Identification and
Recognition Technology) for standard library function recognition.
* `Ghidra` - a crowd favorite, Ghidra is a static reverse engineering tool that
disassembles and decompiles a target program using a **recursive descent**
algorithm. Without going too in depth into the differences between
**linear sweep** and **recursive descent** disassembly, Ghidra follows branches
, functions calls, returns, etc. when disassembling a target binary. This is in
contrast to objump which disassembles blocks of bytes which can be assumed to
be valid instructions. **Recursive descent** disassembly provides the reverse
engineer with more context and usually more accurate results for disassembled
programs.
* `Binary Ninja` - created by Vector35 and similar to both IDAPRo and Ghidra,
Binary Ninja is pretty good at providing a high level intermediate language
representation of a target program. Like IDAPro and Ghidra, Binary Ninja allows
reverse engineers to program scripts against its API - this is usually done to
fix known issues with disassembly or provide general quality of life
improvements.

## Pros of Static Reverse Engineering

Reverse engineers can utilize static reverse engineering to gather information
about a target binary without having to load and execute that binary in memory.
This is most useful when a reverse engineer needs to gather information about
a malicious program but wants to avoid having that malicious program execute
its functionality on the reverse engineer's system. Static reverse engineering
is also useful when the reverse engineer does not have the requisite system(s)
to load and execute a target for dynamic analysis. This scenario is common when
inspecting programs extracted from embedded devices or real-time operating
systems. These programs are usually compiled for a different architecture than
the reverse engineer's machine and expect special / custom devices to be
present in order to implement their functionalities. Static reverse engineering
allows analysts the time and ability to deeply inspect a target program and,
with tools like Ghidra and IDAPro, an analyst can generate and analyze control
flow graphs (CFG)s, data dependencies, decompiled code, etc. Finally, reverse
engineers can easily collaborate when conducting static reverse engineering.
The Ghidra Server is a prime example of this, providing change control for a
repository of programs as analysts comment, label, and update listings.

## Cons of Static Reverse Engineering

Try as we might, some things can't be answered by static reverse engineering.
Often times, if the author of a program intends upon protecting their
intellectual property - or if the author is a malware developer - programs can
come packed / compressed. These usually contain a decoder stub that will
unpack the real program to be executed into the process memory. A number of
unpacking programs exist that help reverse engineers get back to statically
analyzing their targets, however, it's possible a paranoid malware author
could invent their own packer / unpacker with custom encryption, etc.

This increase in sophistication requires reverse engineers to conduct deeper
inspection on custom unpacking algorithms written by malware authors. Luckily,
tools like Ghidra, Binary Ninja, and IDAPro provide scripting interfaces that
allow reverse engineers to write and execute custom scripts. This enables
reverse engineers with the ability to operate on the disassembly
programatically, making the unpacking of custom packed malware more feasible.
