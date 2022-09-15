---
typora-root-url: ..\..\img
---

## Intel Processor Traces {#sec:secPT}

英特尔处理器跟踪 (PT) 是一项 CPU 功能，它通过以高度压缩的二进制格式对数据包进行编码来记录程序执行情况，该二进制格式可用于重建每个指令上带有时间戳的执行流程。 PT 具有广泛的覆盖范围和相对较小的开销[^1]，通常低于 `5%`。它的主要用途是事后分析和根本原因的性能故障。

### 工作流程

与采样技术类似，PT 不需要对源代码进行任何修改。您只需在支持 PT 的工具下运行程序即可收集痕迹。启用 PT 并启动基准测试后，分析工具开始将跟踪数据包写入 DRAM。

与 LBR 类似，英特尔 PT 通过记录分支来工作。在运行时，每当 CPU 遇到任何分支指令时，PT 都会记录该分支的结果。对于简单的条件跳转指令，CPU 将仅使用 1 位记录它是被占用（`T`）还是未被占用（`NT`）。对于间接呼叫，PT 将记录目的地地址。请注意，无条件分支被忽略，因为我们静态地知道它们的目标。

图@fig:PT_encoding 显示了一个小指令序列的编码示例。 `PUSH`、`MOV`、`ADD` 和 `CMP` 等指令会被忽略，因为它们不会改变控制流。但是，`JE`指令可能会跳转到`.label`，所以需要记录它的结果。稍后有一个间接调用，其目的地址被保存。

![英特尔处理器跟踪编码](/4/PT_encoding.jpg){#fig:PT_encoding width=90%}

在分析时，我们将应用程序二进制文件和收集的 PT 跟踪汇总在一起。 SW解码器需要应用程序二进制文件来重构程序的执行流程。它从入口点开始，然后使用收集的跟踪作为查找参考来确定控制流。图@fig:PT_decoding 显示了解码英特尔处理器跟踪的示例。假设`PUSH`指令是应用程序二进制文件的入口点。然后“PUSH”、“MOV”、“ADD”和“CMP”按原样重建，无需查看编码轨迹。稍后 SW 解码器遇到 `JE` 指令，这是一个条件分支，我们需要查找结果。根据图上的痕迹。 @fig:PT_decoding，`JE` 被占用了（`T`），所以我们跳过下一个 `MOV` 指令并转到 `CALL` 指令。同样，`CALL(edx)` 是一条更改控制流的指令，因此我们在编码跟踪中查找目标地址，即 `0x407e1d8`。以黄色突出显示的指令是在我们的程序运行时执行的。请注意，这是对程序执行的*精确*重构；我们没有跳过任何指令。稍后，我们可以使用调试信息将汇编指令映射回源代码，并获得逐行执行的源代码日志。

![Intel 处理器跟踪解码](/4/PT_decoding.jpg){#fig:PT_decoding width=90%}

### 定时数据包 {#sec:PTtimings}

使用 Intel PT，不仅可以跟踪执行流程，还可以跟踪时序信息。除了保存跳转目的地之外，PT 还可以发出定时数据包。图@fig:PT_timings 提供了时间数据包如何用于恢复指令时间戳的可视化。和前面的例子一样，我们首先看到 `JNZ` 没有被使用，所以我们用时间戳 0ns 更新它和上面的所有指令。然后我们看到 2ns 的时间更新和 `JE` 被采用，所以我们更新它以及 `JE` 上面（和 `JNZ` 下面）的所有指令，时间戳为 2ns。之后，有一个间接调用，但没有附加定时数据包，所以我们不更新时间戳。然后我们看到 100ns 过去了，而 `JB` 没有被占用，所以我们用 102ns 的时间戳更新它上面的所有指令。

![Intel Processor Traces timings](/4/PT_timings.jpg){#fig:PT_timings width=90%}

在图@fig:PT_timings 所示的示例中，指令数据（控制流）非常准确，但时序信息不太准确。 显然，`CALL(edx)`、`TEST` 和 `JB` 指令不会同时发生，但我们没有更准确的时序信息。 拥有时间戳使我们可以将程序的时间间隔与系统中的其他事件对齐，并且很容易与挂钟时间进行比较。 某些实现中的跟踪时序可以通过周期精确模式进一步改进，其中硬件记录正常数据包之间的周期计数（请参阅 [@IntelSDM，第 3C 卷，第 36 章] 中的更多详细信息）。

### Collecting and Decoding Traces

Intel PT traces can be easily collected with the Linux `perf` tool:

```bash
$ perf record -e intel_pt/cyc=1/u ./a.out
```

在上面的命令行中，我们要求 PT 机制在每个周期更新时序信息。 但很可能，它不会大大提高我们的准确性，因为只有在与其他一些控制流数据包配对时才会发送计时数据包（参见 [@sec:PTtimings]）。

采集后，可以通过执行以下命令获取原始的 PT 轨迹：

```重击
$ 性能报告 -D > trace.dump
```

PT 在发出定时数据包之前最多捆绑 6 个条件分支。 自英特尔 Skylake CPU 生成以来，计时数据包的周期计数已从前一个数据包开始。 如果我们然后查看 `trace.dump`，我们可能会看到如下内容：

```
000073b3: 2d 98 8c  TIP 0x8c98     // target address (IP)
000073b6: 13        CYC 0x2        // timing update
000073b7: c0        TNT TNNNNN (6) // 6 conditional branches
000073b8: 43        CYC 0x8        // 8 cycles passed
000073b9: b6        TNT NTTNTT (6)
```

Above we showed the raw PT packets, which are not very useful for performance analysis. To decode processor traces to human-readable form, one can execute:

```bash
$ perf script --ns --itrace=i1t -F time,srcline,insn,srccode
```

Below is the example of decoded traces one might get:

```
timestamp       srcline   instruction      srccode
...
253.555413143:  a.cpp:24  call 0x35c       foo(arr, j);
253.555413143:  b.cpp:7   test esi, esi    for (int i = 0; i <= n; i++)
253.555413508:  b.cpp:7   js 0x1e
253.555413508:  b.cpp:7   movsxd rsi, esi
...
```

上面仅显示了长执行日志中的一小段。在此日志中，__我们跟踪程序运行时执行的每条指令__。我们可以从字面上观察程序所做的每一步。它为进一步分析奠定了非常坚实的基础。

### 用法

以下是 PT 可能有用的一些情况：

1. **分析性能故障**。因为 PT 捕获了整个指令流，所以可以分析在应用程序没有响应的小时间段内发生了什么。更详细的示例可以在 easyperf 博客上的 [文章](https://easyperf.net/blog/2019/09/06/Intel-PT-part3)[^2] 中找到。
2. **事后调试**。 PT 跟踪可以由传统的调试器重放，例如 `gdb`。除此之外，PT 提供调用堆栈信息，即使堆栈损坏[^3]，它*总是*有效。 PT 跟踪可以在远程机器上收集一次，然后离线分析。当问题难以重现或对系统的访问受到限制时，这尤其有用。
3. **内省程序的执行**。
   - 我们可以立即判断某些代码路径是否从未执行过。
   - 多亏了时间戳，可以计算在锁定尝试旋转时等待的时间等。
   - 通过检测特定指令模式来缓解安全性。

### 磁盘空间和解码时间

即使考虑到轨迹的压缩格式，编码数据也会消耗大量磁盘空间。通常，每条指令不到 1 个字节，但是考虑到 CPU 执行指令的速度，它仍然很多。根据工作负载，CPU 以 100 MB/s 的速度对 PT 进行编码是很常见的。解码后的轨迹可能很容易增加十倍（~1GB/s）。这使得 PT 不适用于长时间运行的工作负载。但是即使在大工作量下也可以在短时间内运行它。在这种情况下，用户可以仅在故障发生的时间段内附加到正在运行的进程。或者他们可以使用循环缓冲区，其中新的痕迹将覆盖旧的痕迹，即，总是在最后 10 秒左右有痕迹。

用户可以通过多种方式进一步限制收集。他们可以限制仅在用户/内核空间代码上收集跟踪。此外，还有一个地址范围过滤器，因此可以动态选择加入和退出跟踪以限制内存带宽。这使我们可以只跟踪单个函数甚至单个循环。 [^4]

解码 PT 轨迹可能需要很长时间。在 Intel Core i5-8259U 机器上，对于运行 7 毫秒的工作负载，编码的 PT 跟踪大约需要 1MB。使用 `perf script` 解码此跟踪大约需要 20 秒。 `perf script -F time,ip,sym,symoff,insn` 的解码输出占用约 1.3GB 的磁盘空间。

\personal{Intel PT 应该是性能分析的最终游戏。由于运行时开销低，它是一个非常强大的分析功能。但是，现在（2020 年 2 月），使用 `perf script -F` 和 `+srcline` 或 `+srccode` 解码跟踪变得非常缓慢，不适合日常使用。 Linux perf 的实现可能会得到改进。英特尔 VTune Profiler 对 PT 的支持仍处于试验阶段。}

**References and links**

* 英特尔出版物《处理器跟踪》，网址：[https://software.intel.com/en-us/blogs/2013/09/18/processor-tracing](https://software.intel.com/en-我们/博客/2013/09/18/处理器跟踪）。
* 英特尔® 64 和 IA-32 架构软件开发人员手册 [@IntelSDM，第 3C 卷，第 36 章]。
* 白皮书“硬件辅助指令分析和延迟检测”[@IntelPTPaper]。
* Andi Kleen 关于 LWN 的文章，网址：[https://lwn.net/Articles/648154](https://lwn.net/Articles/648154)。
* 英特尔 PT 微教程，网址：[https://sites.google.com/site/intelptmicrotutorial/](https://sites.google.com/site/intelptmicrotutorial/)。
* simple_pt：Linux 上的简单 Intel CPU 处理器跟踪，URL：
  [https://github.com/andikleen/simple-pt/](https://github.com/andikleen/simple-pt)。
* Linux 内核中的 Intel PT 文档，URL：
  [https://github.com/torvalds/linux/blob/master/tools/perf/Documentation/intel-pt.txt](https://github.com/torvalds/linux/blob/master/tools/perf/文档/intel-pt.txt）。
* 英特尔处理器跟踪的备忘单，网址：[http://halobates.de/blog/p/410](http://halobates.de/blog/p/410)。


[^1]: See more information on overhead in [@IntelPTPaper].
[^2]: Analyze performance glitches with Intel PT - [https://easyperf.net/blog/2019/09/06/Intel-PT-part3](https://easyperf.net/blog/2019/09/06/Intel-PT-part3).
[^3]: Postmortem debugging with Intel PT - [https://easyperf.net/blog/2019/08/30/Intel-PT-part2](https://easyperf.net/blog/2019/08/30/Intel-PT-part2).
[^4]: Cheat sheet for Intel PT - [http://halobates.de/blog/p/410](http://halobates.de/blog/p/410).
