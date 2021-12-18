# Stack buffer overflow

## The stack

Before we talk about overflowing data structures on the stack, let's define
what the stack is.

The stack is a data structure with two principal operations, `push` and `pop`.
The stack follows a **last in, first out (LIFO)** convention meaning the
top-most element of the stack is the first to be removed from the data
structure when a `pop` operation occurs. Newer values are `push`ed to the top
of the stack and cannot be removed until succeeding values are `pop`ed from the
stack. [[1]](#references)

The rest of our discussion is related to the **process stack** of ELF binaries.
Each process has a contiguous segment of memory that is set aside to store
information about the active subroutines of the program. The initial stack
layout provides the process access to the command line arguments and
environment used when executing the program. An example of the initial process
stack can be found below [[2]](#references):

```c
argc            // argument count (int)
argv[0]         // program name (char*)
argv[1]         // program arguments (char*)
...
argv[argc-1]
NULL            // end of arguments (void*)
env[0]          // environment (char*)
...
env[n]
NULL            // end of environment (void*)
```

The stack can be implemented to grow down (towards lower memory addresses) or
up (towards higher memory addresses). Most common stack implementations grow
downwards as data is `push`ed to the stack and this is what we will use for
this discussion. A register called the **stack pointer** is used to track
the last address of the stack, the most recent element `push`ed to the stack.
Many compilers also use a second register, the **frame pointer**, to reference
local variables and parameters passed to functions.

### Stack frames

The stack is used to implement functions for programs. For each function call,
a section of the stack is reserved for the function - a **stack frame**. Below
is some example C code that we will use for the rest of our stack frame
discussion:

```c
int my_function(int a, int b) {
    char buffer[32];
    return a + b;
}

int main(int argc, char** argv) {
    my_function(1, 2);
    return 0;
}
```

Example assembly language output for the call to `my_function()` could be:

```c
push    2               // push arguments in reverse
push    1
call    my_function     // push instruction pointer to stack and jump
                        // to beginning of my function
```

The above assembly code follows the `cdecl` calling convention, we'll also use
this calling convention for the rest of our discussion. The above assembly code
showcases how the arguments for the callee function, `my_function()`, are being
passed to the function by the caller, `main`, using the stack. The `call`
instruction `push`es the instruction pointer onto the stack. This will be used
by a `ret` instruction to return to the caller. [[3]](#references)

Entering `my_function()`, we'll see the **function prologue** setting up the
stack frame. Here's an example of what this would look like:

```c
push    rbp         // save the frame pointer of the caller to the stack
mov     rbp, rsp    // set the new frame pointer
sub     rsp, 64     // make space for char buffer[32]
```

Below is an example of what the stack frame would look like for `my_function()`
:

```c
0xdeadbeefcafe0000      buffer[0]               // rsp
...
0xdeadbeefcafe0040      buffer[31]              // end of buffer
0xdeadbeefcafe0048      saved frame pointer     // rbp
0xdeadbeefcafe0050      return address
0xdeadbeefcafe0058      a
0xdeadbeefcafe0060      b
```

Functions use the `rbp` and relative addressing to reference local variables
and parameters passed to the function.

Lastly, we have the **function epilogue** which reverses the actions of the
prologue, restoring the caller's stack frame and returning to the caller. An
example function prologue follows:

```c
leave   // mov rsp, rbp; pop rbp;
ret
```

The `leave` instruction moves the `rsp` back to where it was before we entered
the function, directly after the caller executed the `call` instruction. The
`rbp` is also restored so the caller can correctly access the contents of its
stack frame when it resumes execution. The `ret` instruction sets the program
counter to the **return address** now contained at the top of the stack.
[[4]](#references)

## Buffer overflows

Finally, we can talk about stack buffer overflows and how they can be used to
hijack the execution of a process. Stack buffer overflow vulnerabilities are
a child of the out-of-bounds write weakness and are a condition in which a
buffer being overwritten is allocated on the stack.
[[5]](#references)[[6]](#references)
Provided below is some example C code that contains a stack buffer overflow
vulnerability:

```c
#define MAX_SIZE 64

int main(int argc, char** argv) {
    char buffer[MAX_SIZE] = {0};
    strcpy(buffer, argv[1]);

    return 0;
}
```

In this example, the size of `argv[1]` is not being checked prior to writing
its contents to the stack buffer, `char buffer[MAX_SIZE]`. If `argv[1]` is
greater than `MAX_SIZE`, the `strcpy()` operation will copy the contents of
`argv[1]` to `buffer` but will also overwrite the bytes directly after the
`buffer` variable on the stack.

Referencing the **stack frame** layout examples provided earlier, we can see
that this out-of-bounds write on the stack can lead to the corruption of
sensitive stack data, specifically the **saved frame pointer** and the **return
address**.

### So how can this lead to arbitrary code execution?

Stack information that is usually targeted by an attacker to gain code
execution is the return address. Overwriting this, an attacker can redirect
code execution to any location in memory in which the attacker has write
access. Historically, the first in-depth article that demonstrates using a
stack buffer overflow to gain arbitrary code execution is
[Smashing The Stack For Fun And Profit](#references) by **Aleph One**.

As smashing the stack became more popular, various mitigations were implemented
to protect sensitive information on the stack from being overwritten, e.g.
stack canaries. Another mitigation involves setting permissions for segments of
memory within a process, and setting the permissions of the stack to be
read/write only - preventing the execution of shellcode stored in the stack.
These mitigations led to the creation of the `ret2*` techniques and return
oriented programming (ROP).

Other important stack information that can be targeted is the stack pointer and
the saved frame pointer. These values are used to conduct relative addressing
of variables within the stack frame, their corruption can be leveraged to
complete a "write-what-where" condition. Corruption of the stack pointer and
saved frame pointer can also be used to conduct a stack pivot, allowing the
attacker to control the location and contents of the stack frame used by the
calling function when it resumes execution.

## References

1. [https://web.archive.org/web/20130225162302/http://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Mips/stack.html](https://web.archive.org/web/20130225162302/http://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Mips/stack.html)
2. [http://asm.sourceforge.net/articles/startup.html](http://asm.sourceforge.net/articles/startup.html)
3. [https://www.agner.org/optimize/calling_conventions.pdf](https://www.agner.org/optimize/calling_conventions.pdf)
4. [http://jdebp.eu./FGA/function-perilogues.html](http://jdebp.eu./FGA/function-perilogues.html)
5. [https://cwe.mitre.org/data/definitions/787.html](https://cwe.mitre.org/data/definitions/787.html)
6. [https://cwe.mitre.org/data/definitions/121.html](https://cwe.mitre.org/data/definitions/121.html)
7. [http://phrack.org/issues/49/14.html](http://phrack.org/issues/49/14.html)
