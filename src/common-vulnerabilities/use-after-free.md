# Use-after-free (UAF)

The **Use-after-free** vulnerability can be defined as the use of heap
allocated memory after it has been freed or deleted. [[1]](#references) This
can lead to undefined behavior by the program and is commonly used by attackers
to implement a **Write-what-where** condition. [[2]](#references)

**Double frees** and UAF vulnerabilities are closely related, and double frees
can be used to duplicate chunks in the fastbin, eventually allowing the
attacker to acquire a pointer to free memory. [[3]](#references)
**Heap overflows** can also lead to a UAF vulnerability, given the right
conditions. This is discussed further in the exploitation portion of
[Single Byte Overflows](./single-byte-overflows.md) as we leak `glibc`
addresses from an unsortedbin chunk using our overlapping chunk.

Provided below is an example of a UAF from OWASP.org [[4]](#references):

```c
#include <stdio.h>
#include <unistd.h>

#define BUFSIZER1   512
#define BUFSIZER2   ((BUFSIZER1/2) - 8)

int main(int argc, char **argv) {
	char *buf1R1;
	char *buf2R1;
	char *buf2R2;
	char *buf3R2;

	buf1R1 = (char *) malloc(BUFSIZER1);
	buf2R1 = (char *) malloc(BUFSIZER1);

	free(buf2R1);

	buf2R2 = (char *) malloc(BUFSIZER2);
	buf3R2 = (char *) malloc(BUFSIZER2);

	strncpy(buf2R1, argv[1], BUFSIZER1-1);
	free(buf1R1);
	free(buf2R2);
	free(buf3R2);
}
```

The following sections, [Fastbin Dup](./fastbin-dup.md) and
[Unsortedbin Attack](./unsortedbin-attack.md), demonstrate how UAF
vulnerabilities can be leveraged to gain arbitrary code execution.

## References

1. [https://cwe.mitre.org/data/definitions/416.html](https://cwe.mitre.org/data/definitions/416.html)
2. [https://cwe.mitre.org/data/definitions/123.html](https://cwe.mitre.org/data/definitions/123.html)
3. [https://cwe.mitre.org/data/definitions/415.html](https://cwe.mitre.org/data/definitions/415.html)
5. [https://owasp.org/www-community/vulnerabilities/Using_freed_memory](https://owasp.org/www-community/vulnerabilities/Using_freed_memory)
