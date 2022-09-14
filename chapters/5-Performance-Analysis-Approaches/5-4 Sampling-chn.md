---
typora-root-url: ..\..\img
---

## Sampling {#sec:profiling}

抽样是进行性能分析最常用的方法。人们通常将其与在程序中查找热点联系起来。一般而言，采样给出了以下问题的答案：代码中的哪个位置对某些性能事件的贡献最大。如果我们考虑寻找热点，问题可以重新表述为代码中的哪个位置消耗最多的 CPU 周期。人们经常使用术语“Profiling”来表示技术上所谓的采样。根据 [Wikipedia](https://en.wikipedia.org/wiki/Profiling_(computer_programming))[^1]，分析是一个更广泛的术语，包括各种收集数据的技术，包括中断、代码检测, 和 PMC。

这可能会让人感到意外，但可以想象的最简单的采样分析器是调试器。实际上，您可以通过 a) 在调试器下运行程序，b) 每 10 秒暂停程序，以及 c) 记录它停止的地方来识别热点。如果您多次重复 b) 和 c)，您将构建样本集合。您停止最多的代码行将是程序中最热的地方。 [^6] 当然，这是对真实分析工具如何工作的过度简化的描述。现代分析器每秒能够收集数千个样本，这可以非常准确地估计基准中最热的地方。

与使用调试器的示例一样，每次捕获新样本时都会中断分析程序的执行。在中断时，分析器收集程序状态的快照，构成一个样本。为每个样本收集的信息可能包括中断时执行的指令地址、寄存器状态、调用堆栈（参见 [@sec:secCollectCallStacks]）等。收集的样本存储在数据收集文件中，可以进一步使用显示调用图、程序中最耗时的部分以及统计上重要的代码段的控制流。

### 用户模式和基于硬件事件的采样

采样可以在 2 种不同的模式下执行，使用用户模式或基于硬件事件的采样 (EBS)。用户模式采样是一种纯软件方法，它将代理库嵌入到分析的应用程序中。代理为应用程序中的每个线程设置操作系统计时器。计时器到期后，应用程序会收到由收集器处理的“SIGPROF”信号。 EBS 使用硬件 PMC 来触发中断。特别是使用了 PMU 的计数器溢出特性，我们将在下一节中讨论。[@IntelVTuneGuide]

用户模式采样只能用于识别热点，而 EBS 可用于涉及 PMC 的其他分析类型，例如缓存未命中采样、TMA（参见 [@sec:TMA]）等。

用户模式采样比 EBS 产生更多的运行时开销。当采样使用默认间隔 10ms 时，用户模式采样的平均开销约为 5%。在 1ms 的采样间隔上，基于事件的采样的平均开销约为 2%。通常，EBS 更准确，因为它允许收集更高频率的样本。但是，用户模式采样生成的要分析的数据要少得多，处理它所需的时间也更少。

### Finding Hotspots

在本节中，我们将讨论将 PMC 与 EBS 结合使用的场景。图@fig:Sampling 说明了 PMU 的计数器溢出特性，用于触发性能监控中断（PMI）。

![使用性能计数器进行采样](/2/SamplingFlow.png){#fig:Sampling width=90%}

一开始，我们配置我们想要采样的事件。识别热点意味着知道程序大部分时间都花在哪里。所以循环采样是非常自然的，它是许多分析工具的默认设置。但这不一定是严格的规则；我们可以对我们想要的任何表演事件进行采样。例如，如果我们想知道程序经历最多 L3-cache 未命中次数的位置，我们将对相应的事件进行采样，即“MEM_LOAD_RETIRED.L3_MISS”。

准备工作完成后，我们启用计数并放开基准测试。我们将 PMC 配置为计数周期，因此它将在每个周期递增。最终，它会溢出。在计数器溢出时，HW 将提高 PMI。分析工具配置为捕获 PMI，并具有用于处理它们的中断服务例程 (ISR)。在这个例程中，我们做了多个步骤：首先，我们禁用计数；之后，我们记录计数器溢出时CPU执行的指令；然后，我们将计数器重置为“N”并恢复基准测试。

现在，让我们回到值“N”。使用这个值，我们可以控制我们想要获得新中断的频率。假设我们想要更精细的粒度，每 100 万条指令就有一个样本。为了实现这一点，我们可以将计数器设置为“-1”百万，这样它就会在每 100 万条指令后溢出。该值通常称为“采样后”值。

我们多次重复该过程以建立足够的样本集合。如果我们稍后聚合这些样本，我们可以构建程序中最热位置的直方图，如下面的 Linux `perf record/report` 输出中所示。这为我们提供了按降序排序（热点）的程序功能的开销细分。 [Phoronix 测试套件](https://www.phoronix-test-suite.com/)[^] 中采样 [x264](https://openbenchmarking.org/test/pts/x264)[^7] 基准的示例8]如下图所示：

```bash
$ perf record -- ./x264 -o /dev/null --slow --threads 8 Bosphorus_1920x1080_120fps_420_8bit_YUV.y4m
$ perf report -n --stdio
# Samples: 364K of event 'cycles:ppp'
# Event count (approx.): 300110884245
# Overhead  Samples  Shared Object   Symbol
# ........  .......  .............   ........................................
#
     6.99%    25349    x264          [.] x264_8_me_search_ref
     6.70%    24294    x264          [.] get_ref_avx2
     6.50%    23397    x264          [.] refine_subpel
     5.20%    18590    x264          [.] x264_8_pixel_satd_8x8_internal_avx2
     4.69%    17272    x264          [.] x264_8_pixel_avg2_w16_sse2
     4.22%    15081    x264          [.] x264_8_pixel_avg2_w8_mmx2
     3.63%    13024    x264          [.] x264_8_mc_chroma_avx2
     3.21%    11827    x264          [.] x264_8_pixel_satd_16x8_internal_avx2
     2.25%     8192    x264          [.] rd_cost_mb
	 ...
```

然后，我们自然想知道热点列表中出现的每个函数中的热门代码。 要查看内联函数的分析数据以及为特定源代码区域生成的汇编代码，需要使用调试信息（`-g` 编译器标志）构建应用程序。 用户可以使用 `-gline-tables-only` 选项将调试信息的数量减少 [^4] 到符号的行号，因为它们出现在源代码中。 像 Linux `perf` 这样没有丰富图形支持的工具，将源代码与生成的程序集混合在一起，如下所示：

```bash
# snippet of annotating source code of 'x264_8_me_search_ref' function
$ perf annotate x264_8_me_search_ref --stdio
Percent | Source code & Disassembly of x264 for cycles:ppp 
----------------------------------------------------------
  ...
        :                 bmx += square1[bcost&15][0]; <== source code
  1.43  : 4eb10d:  movsx  ecx,BYTE PTR [r8+rdx*2]      <== corresponding
                                                           machine code
        :                 bmy += square1[bcost&15][1];
  0.36  : 4eb112:  movsx  r12d,BYTE PTR [r8+rdx*2+0x1]
        :                 bmx += square1[bcost&15][0];
  0.63  : 4eb118:  add    DWORD PTR [rsp+0x38],ecx
        :                 bmy += square1[bcost&15][1];
  ...
```

大多数具有图形用户界面 (GUI) 的分析器，例如英特尔 VTune Profiler，可以并排显示源代码和相关的程序集，如图 @fig:SourceCode_View 所示。

![Intel® VTune™ Profiler source code and assembly view for [x264](https://openbenchmarking.org/test/pts/x264) benchmark.](/2/Vtune_source_level.png){#fig:SourceCode_View width=90%}

### Collecting Call Stacks {#sec:secCollectCallStacks}

通常在采样时，我们可能会遇到程序中最热的函数被多个调用者调用的情况。图@fig:CallStacks 显示了这种情况的一个示例。分析工具的输出可能显示函数 `foo` 是程序中最热门的函数之一，但如果它有多个调用者，我们想知道其中哪个调用者调用 `foo` 的次数最多。对于具有诸如`memcpy`或`sqrt`之类的库函数出现在热点中的应用程序来说，这是一种典型情况。要了解为什么某个特定函数会出现热点，我们需要知道程序的控制流图 (CFG) 中的哪条路径导致了它。

![控制流图：热函数“foo”有多个调用者。](/2/CallStacksCFG.png){#fig:CallStacks width=50%}

分析 `foo` 的所有调用者的逻辑可能非常耗时。我们只想关注那些导致 `foo` 显示为热点的调用者。换句话说，我们想知道程序CFG中最热的路径。分析工具通过在收集性能样本时捕获进程的调用堆栈以及其他信息来实现这一点。然后，所有收集的堆栈都被分组，让我们可以看到导致特定功能的最热路径。

可以通过三种方法在 Linux `perf` 中收集调用堆栈：

1. 帧指针（`perf record --call-graph fp`）。需要使用 `--fnoomit-frame-pointer` 构建二进制文件。从历史上看，帧指针（`RBP`）被用于调试，因为它允许我们在不从堆栈中弹出所有参数的情况下获取调用堆栈（堆栈展开）。帧指针可以立即告诉返回地址。但是，它仅为此目的消耗一个寄存器，因此价格昂贵。它也用于分析，因为它可以实现廉价的堆栈展开。
2. DWARF 调试信息（`perf record --call-graph dwarf`）。需要使用 DWARF 调试信息 `-g` (`-gline-tables-only`) 构建二进制文件。
3. Intel Last Branch Record (LBR) 硬件特性`perf record --call-graph lbr`。不像前两种方法那样深入的调用图。在 [@sec:lbr] 中查看有关 LBR 的更多信息。

下面是使用 Linux `perf` 在程序中收集调用堆栈的示例。通过查看输出，我们知道 55% 的时间 `foo` 是从 `func1` 调用的。我们可以清楚地看到 `foo` 的调用者之间的开销分布，现在可以将注意力集中在程序 CFG 中最热的边缘上。

```bash
$ perf record --call-graph lbr -- ./a.out
$ perf report -n --stdio --no-children
# Samples: 65K of event 'cycles:ppp'
# Event count (approx.): 61363317007
# Overhead       Samples  Command  Shared Object     Symbol
# ........  ............  .......  ................  ......................
    99.96%         65217  a.out    a.out             [.] foo
            |
             --99.96%--foo
                       |
                       |--55.52%--func1
                       |          main
                       |          __libc_start_main
                       |          _start
                       |
                       |--33.32%--func2
                       |          main
                       |          __libc_start_main
                       |          _start
                       |
                        --11.12%--func3
                                  main
                                  __libc_start_main
                                  _start
```

使用 Intel Vtune Profiler 时，可以通过在配置分析时选中相应的“收集堆栈”框来收集调用堆栈数据[^2]。使用命令行界面时指定 `-knob enable-stack-collection=true` 选项。

\personal{收集调用堆栈的机制是非常重要的理解。我见过一些不熟悉这个概念的开发人员尝试使用调试器来获取这些信息。他们通过中断程序的执行并分析调用堆栈来做到这一点（如 `gdb` 调试器中的 `backtrace` 命令）。开发人员应该允许分析工具来完成这项工作，这会更快并提供更准确的数据。}

### 火焰图

可视化分析数据和程序中最常见的代码路径的一种流行方法是使用火焰图。它允许我们查看哪些函数调用占用了最大的执行时间。图@fig:FlameGraph 显示了 [x264](https://openbenchmarking.org/test/pts/x264) 基准测试的火焰图示例。从上面提到的火焰图我们可以看到，执行时间最多的路径是`x264 -> threadpool_thread_internal -> slices_write -> slice_write -> x264_8_macroblock_analysis`。原始输出是交互式的，允许我们放大特定的代码路径。火焰图由 Brendan Gregg 开发的 [开源脚本](https://github.com/brendangregg/FlameGraph)[^9] 生成。还有其他能够发出火焰图的工具，也许 KDAB [Hotspot](https://github.com/KDAB/hotspot)[^11] 是最受欢迎的替代品。

![A Flame Graph for [x264](https://openbenchmarking.org/test/pts/x264) benchmark.](/2/Flamegraph.jpg){#fig:FlameGraph width=90%}

[^1]: Profiling(wikipedia) - [https://en.wikipedia.org/wiki/Profiling_(computer_programming)](https://en.wikipedia.org/wiki/Profiling_(computer_programming)).
[^2]: See more details in [Intel® VTune™ Profiler User Guide](https://software.intel.com/content/www/us/en/develop/documentation/vtune-help/top/analyze-performance/hardware-event-based-sampling-collection/hardware-event-based-sampling-collection-with-stacks.html).
[^4]: If a user doesn't need full debug experience, having line numbers is enough for profiling the application. There were cases when LLVM transformation passes incorrectly, treated the presence of debugging intrinsics, and made wrong transformations in the presence of debug information.
[^6]: This is an awkward way, though, and we don't recommend doing this. It's just to illustrate the concept.
[^7]: x264 benchmark - [https://openbenchmarking.org/test/pts/x264](https://openbenchmarking.org/test/pts/x264).
[^8]: Phoronix test suite - [https://www.phoronix-test-suite.com/](https://www.phoronix-test-suite.com/).
[^9]: Flame Graphs by Brendan Gregg: [https://github.com/brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph). See more details about all the features on Brendan's dedicated web page: [http://www.brendangregg.com/flamegraphs.html](http://www.brendangregg.com/flamegraphs.html).
[^11]: Hotspot profiler by KDAB - [https://github.com/KDAB/hotspot](https://github.com/KDAB/hotspot).
