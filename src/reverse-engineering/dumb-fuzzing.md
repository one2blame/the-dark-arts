# Dumb fuzzing

## Foreword

Before we begin our discussion on fuzzing techniques, note that the terms and
definitions used by different fuzzing publications, tools, etc. are not
congruent. Currently, there doesn't seem to be a standard language when
discussing ideas and concepts related to fuzzing - every blog or article is
using terms and definitions differently.

Because of this, all of the language we will use for this and future
discussions on fuzzing will be derived from [[1]](#references). It's an
excellent resource that pulls together the most popular fuzzing tools and
publications and attempts to standardize our understanding of the fuzzing
process as well as all of the terms and techniques associated with fuzzing.

## Model-less fuzzing (dumb fuzzing)

Fuzzers are used to generate input for **programs under test (PUTs)**,
attempting to find inputs that crash the program - a crash being some sort of
unrecoverable state like a segmentation fault. Being programmers ourselves, we
understand that most programs have vulnerabilities in only very specifc
sections of code, requiring specific inputs to traverse an execution path that
reaches the vulnerable code.

**Model-less fuzzers**, or dumb fuzzers, generate input data for PUTs without
any knowledge of the structure required for the data expected by the PUT. A
good example provided by [[1]](#references) is the fuzzing of an MP3 player
application. MP3 files have a particular structure that needs to be met and, if
the MP3 player application is provided an invalid file, it will exit without
executing further code. In this example, unless the model-less fuzzer is
provided with some initial **seed** data, the model-less fuzzer will be unable
to traverse further execution paths until it correctly guesses the data
structure expected by the MP3 player application - this is an infeasible
approach to exploitation.

Model-less fuzzers are **mutation-based**, all of the input they generate for
PUTs is based off of a set of **seeds** provided by the researcher. **Seeds**
are input data that can be correctly ingested by the PUT and allows the fuzzer
to reach sections of code that are usually executed after the input data
is validated. Seeds are usually derived from real-world application use. Once
the model-less fuzzer is provided with enough seeds, the fuzzer will generate
**mutations** of the seeds as test cases for the PUT.

### Bit-Flipping

**Bit-flipping** is a common technique used by model-less fuzzers to mutate
seeds and generate test cases for PUTs. The fuzzer flips a number of bits in
the seed, whether it be random or configurable is determined by the
implementation of the fuzzer. Bit-flipping, model-less fuzzers that allow
user-configuration usually employ a configuration parameter called the
**mutation ratio**, a ratio defined as `K` random bits in an `N-bit` seed:
`K/N`.

### Arithmetic Mutation

**Arithmetic mutation** is a mutation operation where a selected byte sequence
is treated as an integer and simple arithmetic is performed on it by the
fuzzer. For instance, AFL selects a 4-byte value from a seed and then replaces
the integer within that seed with a value `+/-` some random, small number `r`.

### Block-based mutation

**Block-based mutation** involves:

* Inserting a randomly generated block of bytes into a random position in a
seed.
* Deleting a randomly selected block of bytes from a seed.
* Replacing a randomly selected block of bytes with a random value in a seed.
* Generating random permutations for the order of blocks in a seed.
* Resizing seeds by appending random blocks.
* Randomly switching blocks of different seeds.

### Dictionary-based mutation

**Dictionary-based mutation** involves a fuzzer using a set of pre-defined
values with significant semantic weight, for example `0`,`-1`, or format
strings, for mutation.

## Feedback and evaluation

So how does a fuzzer detect the discovery of an interesting input? Model-based
and model-less fuzzers both leverage the use of **bug oracles** for input
evaluation. A **bug oracle** is a part of the fuzzer that determines whether
a given execution of the PUT violates a specific security policy. Usually, bug
oracles are designed to detect segmentation faults but, with the implementation
of sanitizers, fuzzers can also detect unsafe memory accesses, violations of
control flow integrity, and other undefined behaviors.

## Problems with model-less fuzzing

A practical challenge that model-less, mutation-based fuzzing faces is a
program's use of checksum validation, a primary example being the use of a
**cyclic redundancy check (CRC)**. A CRC is an error-detecting code that
ensures that the integrity of the data contained in the input file is preserved
during transmission. If a PUT computes the checksum of an input before parsing
it, it's highly likely that most of the test input provided by the fuzzer will
be rejected.

In this case, model-base fuzzing of the PUT is more likely to succeed, as
showcased in Experiment 2 of [[2]](#references). During the experiment, the
model-based fuzzer had full knowledge of how the input data was to be
structured, including how the checksum was generated. This allowed the fuzzer
to achieve greater code coverage than any of the model-less, mutation-based
fuzzer implementations.

## References

1. [https://arxiv.org/pdf/1812.00140.pdf](https://arxiv.org/pdf/1812.00140.pdf)
2. [https://www.ise.io/wp-content/uploads/2019/11/cmiller_defcon2007.pdf](https://www.ise.io/wp-content/uploads/2019/11/cmiller_defcon2007.pdf)
