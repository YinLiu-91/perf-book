---
typora-root-url: ..\..\img
---

## Tracing

跟踪在概念上与检测非常相似，但略有不同。代码检测假设用户可以编排其应用程序的代码。另一方面，跟踪依赖于程序外部依赖项的现有检测。例如，`strace` 工具允许我们跟踪系统调用，可以被视为 Linux 内核的工具。英特尔处理器跟踪（参见 [@sec:secPT]）允许记录程序执行的指令，并且可以被视为 CPU 的检测。可以从预先进行适当检测且不会更改的组件中获取跟踪。跟踪通常用作黑盒方法，用户无法修改应用程序的代码，但他们希望了解程序在幕后所做的事情。

[@lst:strace] 中演示了使用 Linux `strace` 工具跟踪系统调用的示例。此清单显示了运行 `git status` 命令时的前几行输出。通过使用 `strace` 跟踪系统调用，可以知道每个系统调用的时间戳（最左边的列）、它的退出状态以及每个系统调用的持续时间（在尖括号中）。

清单：使用 strace 跟踪系统调用。
		
~~~~ {#lst:strace .bash}
$ strace -tt -T -- git status
17:46:16.798861 execve("/usr/bin/git", ["git", "status"], 0x7ffe705dcd78 
                  /* 75 vars */) = 0 <0.000300>
17:46:16.799493 brk(NULL)               = 0x55f81d929000 <0.000062>
17:46:16.799692 access("/etc/ld.so.nohwcap", F_OK) = -1 ENOENT 
                  (No such file or directory) <0.000063>
17:46:16.799863 access("/etc/ld.so.preload", R_OK) = -1 ENOENT 
                  (No such file or directory) <0.000074>
17:46:16.800032 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3 
                  <0.000072>
17:46:16.800255 fstat(3, {st_mode=S_IFREG|0644, st_size=144852, ...}) = 0 
                  <0.000058>
17:46:16.800408 mmap(NULL, 144852, PROT_READ, MAP_PRIVATE, 3, 0) 
                  = 0x7f6ea7e48000 <0.000066>
17:46:16.800619 close(3)                = 0 <0.000123>
...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

跟踪的开销很大程度上取决于我们试图跟踪的内容。例如，如果我们跟踪几乎从不进行系统调用的程序，在 `strace` 下运行它的开销将接近于零。相反，如果我们跟踪严重依赖系统调用的程序，开销可能会非常大，比如 100x [^1]。此外，跟踪可以生成大量数据，因为它不会跳过任何样本。为了弥补这一点，跟踪工具提供了仅针对特定时间片或代码段过滤集合的方法。

通常，类似于仪器的跟踪用于探索系统中的异常。例如，您可能想了解在 10 秒无响应期间应用程序中发生了什么。分析不是为此而设计的，但是通过跟踪，您可以看到导致程序无响应的原因。例如，使用 Intel PT（参见 [@sec:secPT]），我们可以重构程序的控制流并准确了解执行了哪些指令。

跟踪对于调试也非常有用。它的基本性质支持基于记录的跟踪的“记录和重放”用例。此类工具之一是 Mozilla [rr](https://rr-project.org/)[^2] 调试器，它可以记录和重放进程，允许向后单步执行等等。大多数跟踪工具都能够用时间戳装饰事件（参见 [@lst:strace] 中的示例），这使我们能够与当时发生的外部事件建立关联。 IE。当我们观察到程序中的故障时，我们可以查看应用程序的痕迹，并将此故障与当时整个系统中发生的情况相关联。

[^1]: An article about `strace` by B. Gregg - [http://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html](http://www.brendangregg.com/blog/2014-05-11/strace-wow-much-syscall.html)

[^2]: Mozilla rr debugger - [https://rr-project.org/](https://rr-project.org/).