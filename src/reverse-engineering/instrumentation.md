# Instrumentation

Before we begin a discussion on instrumentation, let's define the different
types of fuzzing because they can be categorized based upon the amount of
instrumentation they leverage.

### Black-box fuzzers

**Black-box fuzzers** use fuzzing techniques that do not see the internals of
**programs under test (PUTs)**. Black-box fuzzers make decisions upon the
input/output behavior of the PUT. These types of fuzzers are colloquially
called IO-driven or data-driven in the software testing community.
[[1]](#references)

### White-box fuzzers

**White-box fuzzers** generate test cases by analyzing the internals of a PUT,
exploring the state space of the PUT systematically. White-box fuzzing is also
called dynamic symbolic execution or **concolic testing** (concrete + symbolic)
. What this basically means is that the fuzzer leverages concrete and symbolic
execution simultaneously, using concrete program states to simplify symbolic
constraints. [[1]](#references)

### Grey-box fuzzers

**Grey-box fuzzers** are able to obtain *some* information about a PUT and its
executions. In contrast with white-box fuzzers, grey-box fuzzers only perform
lightweight static analysis on the PUT and gather dynamic information during
program execution such as code coverage - they utilize approximate information
in order to speed up the rate of testing. [[1]](#references)

## Instrumentation

Grey and white-box fuzzers instrument the PUT to gather feedback for each
execution of the program. Black-box fuzzers don't leverage any instrumentation
so they won't be discussed in this section. What is instrumentation?
[[1]](#references) defines two categories:

* **Static program instrumentation** - instrumentation of a PUT before it is
executed. This is usually done at compile time on either source code or an
intermediate representation. An good example is how AFL instruments every
branch instruction of a PUT in order to compute branch coverage. This feedback
allows AFL to choose test cases that are more likely to traverse more or newer
branches of a PUT.
* **Dynamic program instrumentation** - instrumentation of a PUT while the
program is being executed. Dynamic instrumentation provides the advantage of
easily instrumenting dynamically linked libraries because the instrumentation
is performed at runtime. Leveraging dynamic program instrumentation provides a
fuzzer with a more complete picture of code coverage as its able to obtain
information from external libraries.

## So how does this aid fuzzers?

As stated earlier, instrumentation provides execution feedback to grey and
white-box fuzzers, providing the fuzzers with information useful for making
decisions on which seeds to prioritize for the generation of input data.
Fuzzers leverage feedback from instrumented PUTs to generate statistics and
track which portions of the code have been traversed and which execution paths
are most likely to produce a crash.

Instrumentation also aids fuzzers through in-memory fuzzing.
**In-memory fuzzing** is when a fuzzer only tests a portion of the PUT without
respawning the process for each fuzz iteration. This is often useful for large
applications that use a GUI and often need time to execute a significant amount
of initialization code. The fuzzer can take a memory snapshot of the PUT right
after it has been initialized and then utilize this memory snapshot to begin
the fuzzing campaign. Instrumentation used in this manner increases the speed
in which a fuzzer can execute test cases to find crashing input.

## References

1. [https://arxiv.org/pdf/1812.00140.pdf](https://arxiv.org/pdf/1812.00140.pdf)
