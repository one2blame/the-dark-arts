# Race conditions

Before we explain how **race conditions** can be used for exploitation, let's
first understand race conditions. Like with everything, there's a Common
Weakness Enumeration (CWE) entry for this vulnerability. The formal name for a
race condition is
[Concurrent Execution using Shared Resource with Improper Synchronization](#references).
What this essentially means is that the vulnerable program is leveraging
parallelism, executing sections of code concurrently. Concurrent operations
exist in just about every modern application as it's much more effecient than
executing code sequentially and makes better use of available operating system
resources.

The problem in this situation, as stated in the title, is that the use of
resources across execution contexts is improperly synchronized. If you've dealt
with multi-processing or multi-threading before, you'll understand the need to
protect shared resources by enforcing atomicity and exclusivity. Two or more
execution contexts accessing or modifying a shared resource without the proper
use of synchronization mechanisms can lead to undefined behavior - and
undefined behavior can lead to program exploitation.

In the following sections, we'll discuss a few vulnerabilities that are
children of race conditions, with some examples on how they can be exploited.

## Signal Handler Race Condition

[Signal handler race conditions](#references) can occur when signal handlers
registered for a process use non-reentrnant functions.
[Reentrancy](#references) is a property of a function or subroutine where
multiple invocations can safely run concurrently. Reentrant functions can be
interrupted in the middle of their execution and then safely re-entered
before previous invocations complete execution.

Non-reentrant functions that are commonly involved with this vulnerability
include `malloc()` and `free()` because they may use global or static data
structures for managing memory. This vulnerability manifests when a procedure
used for signal handling is registered for multiple signals and accesses the
shared resources during the handling of the signal. Below is some example code
that demonstrates this vulnerability, written by Michal Zalewski in the article
, [Delivering Signals for Fun and Profit](#references):

```c
/*********************************************************
 * This is a generic verbose signal handler - does some  *
 * logging and cleanup, probably calling other routines. *
 *********************************************************/

void sighndlr(int dummy) {
  syslog(LOG_NOTICE,user_dependent_data);
  // *** Initial cleanup code, calling the following somewhere:
  free(global_ptr2);
  free(global_ptr1);
  // *** 1 *** >> Additional clean-up code - unlink tmp files, etc <<
  exit(0);
}

  /**************************************************
   * This is a signal handler declaration somewhere *
   * at the beginning of main code.                 *
   **************************************************/

  signal(SIGHUP,sighndlr);
  signal(SIGTERM,sighndlr);

  // *** Other initialization routines, and global pointer
  // *** assignment somewhere in the code (we assume that
  // *** nnn is partially user-dependent, yyy does not have to be):

  global_ptr1=malloc(nnn); 
  global_ptr2=malloc(yyy);

  // *** 2 *** >> further processing, allocated memory <<
  // *** 2 *** >> is filled with any data, etc...     <<
```

In this example, Michael discusses that this code is vulnerable because both
`SIGHUP` and `SIGTERM`'s signal handlers execute `syslog()`, which uses
`malloc()`, and `free()` on the same shared resources. These functions are not
re-entrant because they operate on the same global structures, if one of the
signal handler functions were suspended while the other executed, it's possible
that, once `global_ptr2` is `free()`d, `syslog()` could be executed with
attacker-supplied data, re-using the free chunk and overwriting heap metadata.
Given the right conditions, this could lead to process exploitation.

## Race Condition within a Thread

[This particular example](#references) is pretty straight-forward: two threads
of execution in a program use a resource simultaneously without proper
synchronization, possibly causing the threads to use invalid resources, leading
to an undefined program state.

Below is some example C code from CWE demonstrating this vulnerability:

```c
int foo = 0;
int storenum(int num) {
	static int counter = 0;
	counter++;
	if (num > foo) foo = num;
	return foo;
}
```

Again, pretty straight-foward. If this code executed concurrently using two
different threads, the value of `foo` after this procedure's `return`s is not
deterministic between different invocations of the program.

A more interesting and real-world example of this bug that achieves code
execution is demonstrated in the article,
[Racing MIDI messages in Chrome](#references), by Google Project Zero. Some
background information: Chrome implements a user facing JavaScript API that
conducts inter-process communication (IPC) with a privilieged browser process
that provides brokered access to MIDI (Musical Instrument Digital Interface)
devices. Yeah, it's a surprise to me, too.

Anyways, the author goes to describe two bugs he found in the MIDI manager's
implementation, both vulnerable to a use-after-free (UAF) bug due to a race
condition. Because concurrent operations on objects in the heap are not
properly synchronized between threads, it's possible to coerce one of threads
to execute a virtual function of an already free object in the heap. The author
goes on to provide a proof of concept to obtain code execution using the first
bug, leveraging heap grooming to obtain a desirable heap state before forcing a
series of allocations to overwrite the `vtable` of the free object.

Without an adequate sensitive information disclosure, he is unable to
predicatbly redirect the process to obtain arbitrary code execution, however,
he still demonstrates that he can gain control of the `RIP`.

## Time-of-check Time-of-use (TOCTOU)

[Time-of-check time-of-use (TOCTOU)](#references) is a classic vulnerability
that is a child of the race condition weakness. In summation, the program
checks the state of a resource, our permissions to access the resource, etc.,
before actually using it. The actual use of the resource comes *after* the
check, but what if the state of the resource changes before it's used? If we
check the resource but its properties change before we actually use it, our
program is liable to have undefined behavior.

An interesting example of this vulnerability and how it can be exploited was
covered in the [logrotate](#references) challenge of 35C3CTF 2019. In the
challenge, the `logrotate` binary rotates logs within the `/tmp/log` directory,
owned by the user. The `logrotate` program executes as `root`, but creates
files that are owned by `user:user` in the directory of `/tmp/log`. `logrotate`
executes `stat()` on the `/tmp/log` directory prior to using the directory
to write `/tmp/log/pwnme.log`, making `logrotate` vulnerable to a TOCTOU race
condition.

Armed with this information, the author of the writeup uses `inotify` to detect
when a change occurs for the `/tmp/log` directory. As soon as this happens, the
author creates a symbolic link for `/tmp/log` pointing it to
`/etc/bash_completion.d`. `logrotate` creates a `user` owned `pwnme.log` file
in `/etc/bash_completion.d` where the author writes his reverse-shell callback
shell script to. The next time `root` acquires a session, the reverse-shell
callback script is executed and the attacker is able to acquire a `root` shell.

The CTF challenge I just described is also a good example of the CWE: 
[Symbolic Name not Mapping to Correct Object](#references).

## Conclusion

There are a few other vulnerabilities that are children of the general race
condition CWE such as:

* Context Switching Race Condition
* Race Condition During Access to Alternate Channel
* Permission Race Condition During Resource Copy

While these are also interesting CWEs related to race conditions, the ones we
discussed above have pretty solid examples and demonstrations of real-world
application, which is the primary reason why I covered them.

## References

1. [https://cwe.mitre.org/data/definitions/362.html](https://cwe.mitre.org/data/definitions/362.html)
2. [https://cwe.mitre.org/data/definitions/364.html](https://cwe.mitre.org/data/definitions/364.html)
3. [https://en.wikipedia.org/wiki/Reentrancy_(computing)](https://en.wikipedia.org/wiki/Reentrancy_(computing))
4. [https://lcamtuf.coredump.cx/signals.txt](https://lcamtuf.coredump.cx/signals.txt)
5. [https://cwe.mitre.org/data/definitions/366.html](https://cwe.mitre.org/data/definitions/366.html)
6. [https://googleprojectzero.blogspot.com/2016/02/racing-midi-messages-in-chrome.html](https://googleprojectzero.blogspot.com/2016/02/racing-midi-messages-in-chrome.html)
7. [https://cwe.mitre.org/data/definitions/367.html](https://cwe.mitre.org/data/definitions/367.html)
8. [https://tech.feedyourhead.at/content/abusing-a-race-condition-in-logrotate-to-elevate-privileges](https://tech.feedyourhead.at/content/abusing-a-race-condition-in-logrotate-to-elevate-privileges)
9. [https://cwe.mitre.org/data/definitions/386.html](https://cwe.mitre.org/data/definitions/386.html)
10. [https://cwe.mitre.org/data/definitions/368.html](https://cwe.mitre.org/data/definitions/368.html)
11. [https://cwe.mitre.org/data/definitions/421.html](https://cwe.mitre.org/data/definitions/421.html)
