---
typora-root-url: ..\..\img
---

## Processor Event-Based Sampling {#sec:secPEBS}

基于处理器事件的采样 (PEBS) 是 CPU 中另一个非常有用的功能，它提供了许多不同的方法来增强性能分析。与 Last Branch Record 类似（参见 [@sec:lbr]），PEBS 用于分析程序以捕获每个收集到的样本的额外数据。在 Intel 处理器中，PEBS 功能是在 NetBurst 微架构中引入的。 AMD 处理器上的一个类似功能称为基于指令的采样 (IBS)，从家庭 10h 代内核（代号“巴塞罗那”和“上海”）开始可用。

这组附加数据具有定义的格式，称为 PEBS 记录。当为 PEBS 配置了性能计数器时，处理器会保存 PEBS 缓冲区的内容，然后将其存储在内存中。该记录包含处理器的架构状态，例如通用寄存器（`EAX`、`EBX`、`ESP`等）、指令指针寄存器（`EIP`）、标志寄存器（ `EFLAGS`）等等。 PEBS 记录的内容布局因支持 PEBS 的不同实现而异。有关枚举 PEBS 记录格式的详细信息，请参阅 [@IntelSDM, Volume 3B, Chapter 18.6.2.4 Processor Event-Based Sampling (PEBS)]。 Intel Skylake CPU 的 PEBS 记录格式如图 @fig:PEBS_record 所示。

![PEBS Record Format for 6th Generation, 7th Generation and 8th Generation Intel Core Processor Families. *© Image from [@IntelSDM, Volume 3B, Chapter 18].*](/4/PEBS_record.png){#fig:PEBS_record width=90%}

Users can check if PEBS is enabled by executing `dmesg`:

```bash
$ dmesg | grep PEBS
[    0.061116] Performance Events: PEBS fmt1+, IvyBridge events, 16-deep LBR, full-width counters, Intel PMU driver.
```

Linux `perf` 不会像 LBR[^5] 那样导出原始 PEBS 输出。相反，它处理 PEBS 记录并根据特定需要仅提取数据子集。因此，无法使用 Linux `perf` 访问原始 PEBS 记录的集合。但是，Linux `perf` 提供了一些从原始样本处理的 PEBS 数据，可以通过`perf report -D` 访问这些数据。要转储原始 PEBS 记录，可以使用 [`pebs-grabber`](https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber)[^1] 工具。

PEBS 机制为性能监控带来了许多好处，我们将在下一节中讨论这些好处。

### 精确事件

概要分析中的主要问题之一是查明导致特定性能事件的确切指令。正如 [@sec:profiling] 中所讨论的，基于中断的采样基于对特定性能事件的计数并等待它溢出。当发生溢出中断时，处理器需要一些时间来停止执行并标记导致溢出的指令。这对于现代复杂的乱序 CPU 架构来说尤其困难。

它引入了滑动的概念，定义为导致事件的 IP 与事件被标记的 IP 之间的距离（在 PEBS 记录内的 IP 字段中）。 Skid 使发现指令变得困难，这实际上导致了性能问题。考虑一个具有大量缓存未命中和热汇编代码的应用程序，如下所示：

```asm
; load1 
; load2
; load3
```

分析器可能会将“load3”归为导致大量缓存未命中的指令，而实际上，“load1”是应归咎于的指令。 这通常会给初学者带来很多困惑。 感兴趣的读者可以在 [英特尔开发者专区网站](https://software.intel.com/en-us/vtune-help-hardware-event-skid)[^4] 上了解更多关于此类问题的根本原因。

通过让处理器本身将指令指针（连同其他信息）存储在 PEBS 记录中，可以缓解滑动问题。 PEBS 记录中的“EventingIP”字段指示导致事件的指令。 这需要硬件支持，并且通常仅可用于支持的事件子集，称为“精确事件”。 可以在 [@IntelSDM，第 3B 卷，第 18 章] 中找到特定微架构的精确事件的完整列表。 下面列出了 Skylake 微架构的精确事件：

```
INST_RETIRED.*
OTHER_ASSISTS.*
BR_INST_RETIRED.*
BR_MISP_RETIRED.*
FRONTEND_RETIRED.*
HLE_RETIRED.*
RTM_RETIRED.*
MEM_INST_RETIRED.*
MEM_LOAD_RETIRED.*
MEM_LOAD_L3_HIT_RETIRED.*
```

，其中 `.*` 表示组内的所有子事件都可以配置为精确事件。

TMA 方法（参见 [@sec:TMA]）严重依赖精确事件来定位代码执行效率低下的根源。可以在 [easyperf 博客](https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid)[^2] 上找到使用精确事件来减轻打滑的示例。 Linux `perf` 的用户应在事件中添加 `ppp` 后缀以启用精确标记：

```重击
$ perf record -e cpu/event=0xd1,umask=0x20,name=MEM_LOAD_RETIRED.L3_MISS/ppp -- ./a.exe
```

### 降低采样开销

频繁产生中断并让分析工具本身在中断服务例程中捕获程序状态是非常昂贵的，因为它涉及操作系统交互。这就是为什么某些硬件允许在没有任何中断的情况下自动多次采样到专用缓冲区的原因。仅当专用缓冲区已满时，处理器才会引发中断，并将缓冲区刷新到内存中。这比传统的基于中断的采样具有更低的开销。

当为 PEBS 配置性能计数器时，计数器中的溢出条件将启用 PEBS 机制。在溢出之后的后续事件中，处理器将生成一个 PEBS 事件。在 PEBS 事件中，处理器将 PEBS 记录存储在 PEBS 缓冲区中，清除计数器溢出状态并用初始值重新加载计数器。如果缓冲区已满，CPU 将引发中断。 [@IntelSDM，第 3B 卷，第 18 章]

请注意，PEBS 缓冲区本身位于主内存中，其大小是可配置的。同样，性能分析工具的工作是为 CPU 分配和配置内存区域，以便能够在其中转储 PEBS 记录。

### 分析内存访问 {#sec:sec_PEBS_DLA}

内存访问是许多应用程序性能的关键因素。使用 PEBS，可以收集有关程序中内存访问的详细信息。允许这种情况发生的功能称为数据地址分析。为了提供有关采样加载和存储的更多信息，它利用了 PEBS 工具内的以下字段（参见图 @fig:PEBS_record）：

* 数据线性地址 (0x98)
* 数据源编码 (0xA0)
* 延迟值 (0xA8)

如果性能事件支持数据线性地址 (DLA) 工具，并且已启用，CPU 将转储内存地址和采样内存访问的延迟。记住;此功能不会跟踪所有存储和加载。否则，开销会太大。相反，它对内存访问进行采样，即仅分析 1000 次左右的访问中的一次。您可以自定义每秒需要多少样本。

此 PEBS 扩展最重要的用例之一是检测 [真/假共享](https://en.wikipedia.org/wiki/False_sharing)[^3]，我们将在 [@sec:TrueFalseSharing] 中讨论. Linux `perf c2c` 工具严重依赖 DLA 数据来查找有争议的内存访问，这可能会遇到真/假共享。

此外，在数据地址分析的帮助下，用户可以获得有关程序中内存访问的一般统计信息：

```bash
$ perf mem record -- ./a.exe
$ perf mem -t load report --sort=mem --stdio
# Samples: 656  of event 'cpu/mem-loads,ldlat=30/P'
# Total weight : 136578
# Overhead       Samples  Memory access
# ........  ............  ........................
    44.23%           267  LFB or LFB hit
    18.87%           111  L3 or L3 hit
    15.19%            78  Local RAM or RAM hit
    13.38%            77  L2 or L2 hit
     8.34%           123  L1 or L1 hit
```

From this output, we can see that 8% of the loads in the application were satisfied with L1 cache, 15% from DRAM, and so on.

[^1]: PEBS grabber tool - [https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber](https://github.com/andikleen/pmu-tools/tree/master/pebs-grabber). Requires root access.
[^2]: Performance skid - [https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid](https://easyperf.net/blog/2018/08/29/Understanding-performance-events-skid).
[^3]: False sharing - [https://en.wikipedia.org/wiki/False_sharing](https://en.wikipedia.org/wiki/False_sharing).
[^4]: Hardware event skid - [https://software.intel.com/en-us/vtune-help-hardware-event-skid](https://software.intel.com/en-us/vtune-help-hardware-event-skid).
[^5]: For LBR, Linux perf dumps entire contents of LBR stack with every collected sample. So, it's possible to analyze raw LBR dumps collected by Linux perf.
