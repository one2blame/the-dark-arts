# Coverage-based fuzzing

We've somewhat discussed code-coverage based fuzzing in the previous sections
when talking about grey-box testing or the AFL fuzzer, but here we'll actually
provide a concrete definition, some examples, and references.

**Code-coverage based fuzzing** is a technique that programmers and fuzzers use
in order to uncover as many bugs as possible within a program. In previous
sections we talked about mutation-based fuzzing and generating random input
with dumb fuzzers in an attempt to generate a crash, but those techniques
aren't super helpful if the program never uses all of its code to evaluate our
input. Code-coverage based fuzzing solves this argument, the more code we
execute within a program the more likely we are to find a bug.

## Examples and resources

### The Fuzzing Book

First I'll start with the [The Fuzzing Book's](#references) article on
code-coverage based fuzzing. This article explains code-coverage based fuzzing
in depth and provides examples for Python and C applications. The article
conducts code-coverage based fuzzing for a CGI decoding application, an
application that takes URL encoded strings and reverts them to their original
text.

In this article, the author also takes the time to make the distinction between
black-box fuzzing and white-box fuzzing, and how white-box fuzzing enables
code-coverage based fuzzing. Without [instrumentation](./instrumentation.md), we
aren't able    to conduct code-coverage based fuzzing because the fuzzer doesn't
have the ability to capture which lines of code have been executed by the
**program under test (PUT)**. With instrumentation, however, we can conduct
code-coverage based fuzzing for both program statements and branch
instructions.

In this particular example, the author uses statement-based coverage to acquire
code-coverage fuzzing statistics. Because the decoder application is so small,
the author only needed to run ~40-60 fuzzing iterations to cover all lines of
code in the application.

### Fuzzing with Code Coverage by Example

[This presentation](#references) given by Charlie Miller at Toorcon 2007
provides some good historical context and expands upon the advancements made
towards mutation-based, generation-based, and code-coverage based fuzzing. It's
a thorough explanation with some great examples and pratical exercises for
code-coverage based fuzzing. The most interesting part is its mention of
evolutionary algorithms for code-coverage based fuzzing, and how mutation-based
, model-less fuzzers can utilize the feedback provided by code-coverage based
fuzzing to select more "fit" inputs.

### Fuzzing by Suman Jana

[This presentation](#references) by Suman Jana from Columbia University in the
City of New York, provides a more recent synopsis of code-coverage based
fuzzing. This presentation covers one of the more popular fuzzers in the
research community that uses code-coverage based fuzzing,
**American Fuzzy Lop (AFL)**. AFL is a model-less, mutation-based, grey-box
fuzzer that leverages instrumentation at compile time to conduct branch-based
code-coverage fuzzing. Throughout a fuzzing campaign, AFL will use its
instrumentation to evaluate what branches a particular seed was able to cover.
AFL attempts to generate a pool of seeds that cover as much of the program as
possible, and also trims seeds from the pool to produce the minimum required
input to cover sections of code.

## Evolutionary Algorithms

An **Evolutionary Algorithm (EA)** uses biological evolution mechanisms such as
mutation, recombination, and selection to maintain a **seed pool** of "fit"
input candidates for a fuzzing campaign, and new candidates can be discovered
and added to this seed pool as more data is collected. Most grey-box fuzzers
leverage EAs in tandem with node or branch coverage to generate seeds,
preserving seeds that discover new nodes or branches and trimming seeds until
the minimum amount of input required to cover a branch is discovered. The
implementations of EAs across different grey-box fuzzers varies, but they all
end up achieving similar goals using this feedback mechanism.
[[4]](#references)

## References

1. [https://www.fuzzingbook.org/html/Coverage.html#](https://www.fuzzingbook.org/html/Coverage.html#)
2. [https://www.ise.io/wp-content/uploads/2019/11/cmiller_toorcon2007.pdf](https://www.ise.io/wp-content/uploads/2019/11/cmiller_toorcon2007.pdf)
3. [https://www.cs.columbia.edu/~suman/secure_sw_devel/fuzzing.pdf](https://www.cs.columbia.edu/~suman/secure_sw_devel/fuzzing.pdf)
4. [https://arxiv.org/pdf/1812.00140.pdf](https://arxiv.org/pdf/1812.00140.pdf)
