# Binary diffing

**Binary diffing** or patch diffing is a technique that compares two binaries
derived from the same code with the goal of discovering their differences and
revealing any technical details that weren't covered in the changelogs,
bulletins, or patch notes for an update. Researchers use binary diffing to
compare known-vulnerable binaries to post-patch binaries in an attempt to suss
out how a vulnerability was patched by the author. It's common for researchers
to also find a way around a discovered patch or the patch can also expose other
vulnerabilities within the binary that were previously overlooked.

Vulnerabilities discovered through binary diffing are colloquially termed
**1-day** bugs / vulnerabilities as they are usually discovered the day after a
patch is released. 1-day bugs can be used to exploit and compromise users or
organizations that are slow to adopt the newest security patches.

Binary diffing is also useful for discovering security patches within entire
code cores for products. In the case of
[this Google Project Zero article](#references), the researcher leverages
binary diffing to discover kernel memory disclosure bugs in the Windows
operating system core - bugs that can be useful to defeat kernel ASLR in other
exploits.

## Limitations

Most of the examples online that aim to teach binary diffing use simple
programs with small changes, however, in real-world scenarios binary differs do
face some obstacles. Here are a few examples as listed by an Incibe-Cert
article on binary diffing [[2]](#references):

* **Unrelated differences** - it's common that changes in the compiler or
compiler optimizations can cause small differences to manifest between two
different binaries. A security researcher must be able to identify when a
difference between two binaries is unrelated to a patch.
* **Bugfixes not related to a vulnerability** - sometimes bugfixes are present
in a new version of the binary that are completely unrelated to the
vulnerability of interest.
* **Obfuscation and anti-diffing techniques** - some authors and organizations
purposefully obfuscate and leverage anti-diffing techniques for patches to
prevent researchers from finding vulnerabilities or reverse-engineering patches
to previous vulnerabilities. [This BlackHat 2009 presentation](#references) by
Jeongwook Oh goes into great detail about binary diffing and various
obfuscation and anti-diffing technologies.

## References

1. [https://googleprojectzero.blogspot.com/2017/10/using-binary-diffing-to-discover.html](https://googleprojectzero.blogspot.com/2017/10/using-binary-diffing-to-discover.html)
2. [https://www.incibe-cert.es/en/blog/importance-of-language-binary-diffing-and-other-stories-of-1day](https://www.incibe-cert.es/en/blog/importance-of-language-binary-diffing-and-other-stories-of-1day)
3. [https://www.blackhat.com/presentations/bh-usa-09/OH/BHUSA09-Oh-DiffingBinaries-SLIDES.pdf](https://www.blackhat.com/presentations/bh-usa-09/OH/BHUSA09-Oh-DiffingBinaries-SLIDES.pdf)
