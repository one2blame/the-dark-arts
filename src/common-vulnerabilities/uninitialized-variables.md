# e. How uninitialized variables can be used for exploitation

The [Use of an Uninitialized Variable](#references) is defined as program code
that uses a variable that has not been initialized, leading to unpredictable or
unintended results.

As programmers we know that in languages such as C or C++, unless we
explicitly initialize stack variables before we use them, these stack variables
can contain junk data from the stack - residual data from previous function's
stack frames. If our program makes decisions based upon the contents of an
uninitialized variable, the program might not behave like we expect it to. An
attacker could leverage this vulnerability to control the behavior of the
vulnerable program by ensuring that data they control is written to the
location of the uninitialized variable on the stack.

Below is an example from [[1]](#references) that demonstrates the usage of a
variable that is uninitialized:

```c
int aN, Bn;
switch (ctl) {
    case -1:
        aN = 0;
        bN = 0;
        break;

    case 0:
        aN = i;
        bN = -i;
        break;

    case 1:
        aN = i + NEXT_SZ;
        bN = i - NEXT_SZ;
        break;

    default:
        aN = -1;
        aN = -1;
        break;
}

repaint(aN, bN);
```

## Community research

Halvar Flake from SABRE Security gave a [BlackHat talk](#references) about this
topic back in 2006. He provides a pretty thorough discussion about this
vulnerabilty class as it relates to the stack, and how attackers could conduct
analysis in order to determine if a target with this vulnerability is
exploitable. The author showcases how researchers can graphically analyze the
execution paths of a target binary and calculate the "reachability" of a
subroutine's stack frame from other subroutines - how many words separate the
two stack frames of each subroutine. These tools allow researchers to wargame
the execution paths a target would take if they could gain control of an
uninitialized variable within a target stack frame. It's an interesting read,
however, a bit nebulous since it doesn't go into how to practically abuse this
vulnerability.

Another solid read is [this paper](#references) by researchers from Georgia
Tech and Saarland University. These researchers provide an in-depth discussion
on the feasibility of exploiting uninitialized variable use, and they also
propose the creation of an automated, targeted stack-spraying technique that
provides deterministic values in the kernel stack, proving that attackers can
reliably abuse this vulnerability. Finally, they propose compiler-based
mechanisms to detect and mitigate this vulnerability.

## So what about the heap?

In previous discussions, we've talked about heap spraying - this is also
covered in the paper mentioned above by Georgia Tech researchers. The concept
of using heap spraying to exploit uninitialized variables is similar, the
attacker targets an uninitialized variable in the heap that could likely be
used by the program to make decisions or traverse different paths of execution.
More interesting is the possibility that the program stores, accesses, and
operates on objects/structures in the heap. If the attacker could reliably
spray the heap with forged objects, the attacker could feasibly fool the
program into using a forged object for arbitrary code execution, especially if
the forged object registers pointers to virtual functions that the program
calls.

## References

1. [https://cwe.mitre.org/data/definitions/457.html](https://cwe.mitre.org/data/definitions/457.html)
2. [https://www.blackhat.com/presentations/bh-europe-06/bh-eu-06-Flake.pdf](https://www.blackhat.com/presentations/bh-europe-06/bh-eu-06-Flake.pdf)
3. [https://www-users.cs.umn.edu/~kjlu/papers/tss.pdf](https://www-users.cs.umn.edu/~kjlu/papers/tss.pdf)
