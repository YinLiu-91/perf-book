---
typora-root-url: ..\..\img
---

## Performance Monitoring Unit {#sec:PMU}

每个现代 CPU 都提供了监控性能的方法，这些方法组合到了性能监控单元 (PMU) 中。 它包含可帮助开发人员分析其应用程序性能的功能。 图@fig:PMU 中提供了现代 Intel CPU 中的 PMU 示例。 大多数现代 PMU 都有一组性能监控计数器 (PMC)，可用于收集在程序执行期间发生的各种性能事件。 稍后在 [@sec:counting] 中，我们将讨论如何在性能分析中使用 PMC。 此外，可能还有其他增强性能分析的特性，如 LBR、PEBS 和 PT，整个第 6 章都会专门介绍这些特性。

![Performance Monitoring Unit of a modern Intel CPU.](/uarch/PMU.png){#fig:PMU width=70%}

随着 CPU 设计随着每一代新产品的发展而发展，他们的 PMU 也在不断发展。 可以使用 `cpuid` 命令确定 CPU 中 PMU 的版本，如 [@lst:QueryPMU] 所示。[^1] 每个 Intel PMU 版本的特征，以及对先前版本的更改， 可以在 [@IntelSDM，第 3B 卷，第 18 章] 中找到。

Listing: Querying your PMU

~~~~ {#lst:QueryPMU .bash}
$ cpuid
...
Architecture Performance Monitoring Features (0xa/eax):
      version ID                               = 0x4 (4)
      number of counters per logical processor = 0x4 (4)
      bit width of counter                     = 0x30 (48)
...
Architecture Performance Monitoring Features (0xa/edx):
      number of fixed counters    = 0x3 (3)
      bit width of fixed counters = 0x30 (48)
...
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

### Performance Monitoring Counters {#sec:PMC}

如果我们想象一个处理器的简化视图，它可能看起来像图@fig:PMC 中所示的那样。 正如我们在本章前面所讨论的，现代 CPU 具有缓存、分支预测器、执行管道和其他单元。 当连接到多个单元时，PMC 可以从它们那里收集有趣的统计数据。 例如，它可以计算已经过去了多少时钟周期，执行了多少指令，在此期间发生了多少缓存未命中或分支错误预测，以及其他性能事件。

![Simplified view of a CPU with a performance monitoring counter.](/uarch/PMC.png){#fig:PMC width=60%}

通常，PMC 为 48 位宽，这允许分析工具在不中断程序执行的情况下运行更长时间[^2]。性能计数器是作为模型特定寄存器 (MSR) 实现的硬件寄存器。这意味着计数器的数量及其宽度可能因型号而异，并且您不能依赖 CPU 中相同数量的计数器。例如，您应该始终首先使用“cpuid”之类的工具进行查询。 PMC 可通过“RDMSR”和“WRMSR”指令访问，这些指令只能从内核空间执行。

这很常见，以至于工程师想要计算执行指令的数量和经过的周期数，英特尔 PMU 有专门的 PMC 来收集此类事件。英特尔 PMU 具有固定和可编程 PMC。固定计数器总是测量 CPU 内核内部的相同事物。使用可编程计数器，用户可以选择他们想要测量的内容。每个逻辑核心通常有四个完全可编程的计数器和三个固定功能的计数器。固定计数器通常设置为计算核心时钟、参考时钟和停用的指令（有关这些指标的更多详细信息，请参阅 [@sec:secMetrics]）。

PMU 有大量的性能事件并不罕见。图@fig:PMU 仅显示了可用于在现代 Intel CPU 上监控的所有性能事件的一小部分。不难发现，可用 PMC 的数量远小于性能事件的数量。不可能同时计算所有事件，但分析工具通过在程序执行期间在性能事件组之间多路复用来解决这个问题（参见 [@sec:secMultiplex]）。

英特尔 CPU 性能事件的完整列表可以在 [@IntelSDM，第 3B 卷，第 19 章] 中找到。对于 ARM 芯片，没有那么严格的定义。供应商按照 ARM 架构实现内核，但性能事件差异很大，包括它们的含义和支持的事件。

[^1]: The same information can be extracted from the kernel message buffer by using the `dmesg` command.
[^2]: When the value of PMCs overflows, the execution of a program must be interrupted. SW then should save the fact of overflow.

\sectionbreak
