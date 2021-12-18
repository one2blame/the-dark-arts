# Dynamic reverse engineering

Dynamic reverse engineering is utilized by analysts to derive conclusions that
can't be discovered with static reverse engineering. In order to conduct
dynamic reverse engineering, analysts load and execute the program of
inspection (POI) into memory in a controlled environment. Analysts usually
attempt to recreate the exact environment the POI normally executes in order to
remove any variables that might cause differences for exploit development. The
following are some technologies and techniques used to accomplish dynamic
reverse engineering:

* Containerization - analysts can place POIs into containers using technologies
like Docker. This allows analysts to control the POI's use of system resources
and software packages. Containerization is useful in identifying a POI's
dependencies and enables the recreation of a POI's exploitable environment.
Containers are a excellent choice for quickly creating lightweight, virtualized
environments for dynamic reverse engineering and exploit development.
* Virtualization - virtualization technologies like VMware, VirtualBox, and QEMU
allow analysts to create an entire Guest Operating System running the POI's
intended Operating System, managed by a hypervisor on the Host Operating System
. These virtualization technologies are useful for dynamic reverse engineering
because the analyst can control the Guest Operating System's resources, e.g.
network cards, graphics cards, host dynamic memory, disk memory, etc. Full
virtualization also makes virtualization entirely transparent to the Guest
Operating System - all system calls are translated by the hypervisor to the
host machine without the Guest OS's awareness. This is useful for the
inspection of POI's with more sophisticated virtualization detection
mechanisms.
* QEMU user-mode emulation - this project recevies its own bullet due to its
usefulness for dynamic reverse engineering. QEMU's user-mode emulation through
programs found in the `qemu-user-static` package allows analysts to load and
execute programs compiled for different architectures. Not only that, it
provides an easy interface for changing the POIs environment variables, library
loading behavior, system call tracing, and remote debugging. Knowledge and use
of QEMU's user-mode emulation is a must for analysts attempting to reverse
engineer embedded software.
* Function call hooking - during dynamic reverse engineering, sometimes
programs execute function calls to external libraries expecting specific
responses in order to continue execution. A majority of the time, analysts are
unable to recreate the correct environment in order to correctly service these
external library function calls, thus the program aborts or exits before
executing any interesting behavior. Analysts combat this by writing hooks in C
and compiling shared objects for dynamically-linked POIs, intercepting a POI's
external library function calls and returning correct values so that the
program can continue execution without failure.
* Debugging - with utilities like `gdb` or `windbg`, analysts can examine the
process stack, heap, and registers while a POI executes in process memory. This
deep inspection allows analysts to truly understand how a POI behaves, what
return values it expects from internal and external subroutines, and also opens
a window to extracting packed / encrypted programs from process memory after
decoding.
* Symbolic execution - symbolic execution, sometimes known as symbolic
analysis, is a technique used to derive the inputs to a program that satisfies
a constraint for a specific path of execution. A symbolic execution engine will
analyze a specific slice of the program and determine what variables are used
to reach a target address. From here, a symbolic execution engine can conduct
"concolic execution", a term used to describe a mix of symbolic and concrete
execution wherein the engine determines the satisfiability contraints for a
path of execution and tests different inputs to inspect their side effects.
Symbolic analysis has various security applications and use cases for reverse
engineers, enabling them with the ability to find vulnerabilities, generate
exploits, and conduct full code coverage for malware specimens.

## Pros of Dynamic Reverse Engineering

Reverse engineers can utilize dynamic reverse engineering to answer questions
that can't be satisfied by static reverse engineering. In addition, dynamic
reverse engineering allows analysts to understand the behavior of programs that
have been packed or encrypted, allowing them to unpack or decrypt themselves
prior to being loaded into memory by the decompression / decoder stub of the
program. Using this technique, analysts can dump the program from process
memory onto the disk for further static analysis.

During dynamic reverse engineering, analysts can gradually provide POIs with
necessary resources to inspect behavioral changes. This technique is most
commonly used with programs that receive command and control (C2)
communications from a source on the Internet - allowing POIs to have access to
a network card within a controlled environment.

Dynamic reverse engineering allows analysts to understand the true intent of a
POI, if it wasn't already readily apparent during static analysis. Because most
analysis occurs within a virtual environment, an analyst can execute the
program without risk to lab infrastructure or information - this is useful for
the inspection of malware like ransomware, wipers, worms, etc.

## Cons of Dynamic Reverse Engineering

There will always be inherent risk in executing code that you didn't write
yourself. It's entirely possible that a POI utilizes some undisclosed jailbreak
/ sandbox escape that could cause harm to the analyst's workstation. In
addition, most malware today comes with functionality to detect virtualization
or debugging techniques. Some malware even goes so far as to report an
analyst executing the malware within a sandbox environment - alerting the
malware author to update or change the program distribution to hamper
reverse engineering efforts.

Despite all best efforts, sometimes it will be impossible to recreate the POIs
operating environment. Using dynamic reverse engineering, analysts have to make
assumptions that might not always be correct - this can cause analysts to focus
on unimportant features of the POI or paths of execution that may never take
place in the real environment.

Virtualization for dynamic reverse engineering also has its limitations. A
prime example of this is the Spectre security vulnerability. [[1]](#references)
This vulnerability affects modern processors that perform branch prediction -
attackers can leverage the side effects of speculative execution and branch
misprediction to reveal private data. It would be infeasible for a reverse
engineer to inspect a malware specimen exploiting this vulnerability from a
virtualized environment. Likely, the reverse engineer would have to conduct
debugging and analysis on a physical processor.

# References

1. [https://en.wikipedia.org/wiki/Spectre_(security_vulnerability)](https://en.wikipedia.org/wiki/Spectre_(security_vulnerability))
