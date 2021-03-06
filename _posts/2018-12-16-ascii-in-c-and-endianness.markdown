---
layout: post
title:  "ASCII in C and Endianness"
date:   2018-12-16 12:00:00 -0500
categories: ET_Tour
---
**TL;DR: ASCII characters, when stored in memory in `char` arrays in C, are stored sequentially without regard to endianness.**

[Youtube video related to this post can be found here.](https://youtu.be/WrVJ3JhS6xY)

To demonstrate:

First, we need to check if Address Space Layout Randomization (ASLR) is enabled on the Linux system. ASLR randomizes memory layout as a security measure to discourage certain memory exploits. Disabling it allows us to read memory more easily.

To check if ASLR is enabled, use the following command.
{% highlight shell %}
cat /proc/sys/kernel/randomize_va_space
{% endhighlight %}

* 0 – No randomization. Everything is static.
* 1 – Conservative randomization. Shared libraries, stack, `mmap()`, VDSO and heap are randomized.
* 2 – Full randomization. In addition to elements listed in the previous point, memory managed through `brk()` is also randomized.

To disable ASLR, run the following command in Linux terminal.
{% highlight shell %}
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
{% endhighlight %}

[Source](https://securityetalii.es/2013/02/03/how-effective-is-aslr-on-linux-systems/)

Now let's write our dummy C program:
{% highlight c %}
#include <stdio.h>

int main () {
​	char sad_cold_truth[23] = "BTW, I don't use Arch.";
​	printf("%s\n", sad_cold_truth);
​	return(0);
}
{% endhighlight %}

We save the file as `ETT001.c` on Desktop. Then we use gcc to compile it:
{% highlight shell %}
gcc -o ETT001 ETT001.c -fno-stack-protector -g
{% endhighlight %}

* `gcc` is the compiler program name.
* `-o ETT001` specifies the output name.
* `ETT001.c` specifies the source C file.
* `-fno-stack-protector` disables the stack protector so that assembly and memory can be read more easily.
* `-g` prepares the binary for analysis in gdb.

*Note: If the compiler complains with errors relating to “stray ‘\200’ in program,” it is likely that you used characters that are encoded outside the ASCII range. Quotation marks, tabs, and hyphens are common culprits. Pasting the source code into an IDE should help you identify the offenders.*

Now we are ready to use gdb to crack open the compiled C program.
{% highlight shell %}
gdb ETT001
{% endhighlight %}

We now add a breakpoint at the beginning of line 5.

`(gdb) break ETT001.c:5`

When the program hits this breakpoint, the previous line (below) will have been executed. In other words, the `char` array `sad_cold_truth` will have been allocated and written in memory (and available for our inspection).
{% highlight c %}
char sad_cold_truth[23] = "BTW, I don't use Arch.";
{% endhighlight %}

We then run the program.

`(gdb) run`

The program should automatically break at line 5. Before we can begin examining its memory content, we will need to find out exactly where in memory the program lives. One way to do that is by looking at its stack pointer. On x86_64 machines it is saved in the `%rsp` register after the function is called. We can find its value by dumping registers.

`(gdb) info registers`

In my instance (which may differ from yours), `%rsp` is `0x7fffffffdfb0`.

Now we can examine memory beginning at that location.

`(gdb) x/24xb 0x7fffffffdfb0`

* The first `x` is the examine command in gdb.
* `/24` indicates we wish to examine 24 bytes of memory.
* The second `x` specifies hexadecimal as the output format.
* `b` tells gdb to output the results one byte at a time.
* `0x7fffffffdfb0` is the beginning address to examine.

We then receive the below output:

```
0x7fffffffdfb0:	0x42	0x54	0x57	0x2c	0x20	0x49	0x20	0x64
0x7fffffffdfb8:	0x6f	0x6e	0x27	0x74	0x20	0x75	0x73	0x65
0x7fffffffdfc0:	0x20	0x41	0x72	0x63	0x68	0x2e	0x00	0x00
```

By consulting the ASCII table we see that the `char` array begins at `0x7fffffffdfb0` and terminates with null at `0x7fffffffdfc7`. It is in strict sequential order. **In other words, ASCII characters, when stored in memory in `char` arrays in C, are stored sequentially without regard to endianness.**