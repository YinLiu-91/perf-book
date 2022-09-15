---
typora-root-url: ..\..\img
---

## Top-Down Microarchitecture Analysis {#sec:TMA}

自上而下的微架构分析方法 (TMA) 是一种非常强大的技术，用于识别程序中的 CPU 瓶颈。它是一种强大而正式的方法，即使对于没有经验的开发人员也很容易使用。这种方法最好的部分是它不需要开发人员对系统中的微架构和 PMC 有深入的了解，并且仍然可以有效地发现 CPU 瓶颈。但是，它不会自动修复问题；否则，这本书就不存在了。

在高层次上，TMA 识别出是什么阻碍了程序中每个热点的执行。瓶颈可能与以下四个组件之一有关：前端绑定、后端绑定、退休、错误推测。图@fig:TMA_concept 说明了这个概念。这是有关如何阅读此图表的简短指南。正如我们从 [@sec:uarch] 中了解到的，CPU 中有内部缓冲区，用于跟踪有关正在执行的指令的信息。每当获取和解码新指令时，就会分配这些缓冲区中的新条目。如果在特定的执行周期内没有为指令分配 uop，可能有两个原因：我们无法获取和解码它（前端绑定），或者后端工作负载过重，新 uop 的资源无法被分配（后端绑定）。已分配并计划执行但未停用的 Uop 与错误推测存储桶有关。这种 uop 的一个例子可以是一些推测性地执行但后来被证明在错误的程序路径上并且没有退出的指令。最后，Retiring 是我们希望所有微指令都存在的存储桶，尽管也有例外。非向量化代码的高 Retiring 值可能是用户向量化代码的一个很好的提示（参见 [@sec:Vectorization]）。另一种情况是，我们可能会看到较高的 Retiring 值但整体性能较慢，这可能发生在对非正规浮点值进行操作的程序中，从而使此类操作极其缓慢（请参阅 [@sec:SlowFPArith]）。

![TMA顶级细分背后的概念。 *© 图片来自 [@TMA_ISPASS]*](/4/TMAM_diag.png){#fig:TMA_concept width=90%}

图@fig:TMA_concept 给出了程序中每条指令的细分。但是，分析工作负载中的每条指令肯定是大材小用，当然，TMA 不会这样做。相反，我们通常有兴趣了解是什么阻碍了整个程序。为了实现这一目标，TMA 通过收集特定指标（PMC 的比率）来观察程序的执行情况。基于这些指标，它通过将应用程序与四个高级存储桶之一相关联来表征应用程序。每个高级存储桶都有嵌套类别（参见图@fig:TMA），可以更好地分解程序中的 CPU 性能瓶颈。我们多次运行工作负载[^14]，每次都专注于特定指标并向下钻取，直到我们对性能瓶颈进行更详细的分类。例如，最初，我们收集四个主要存储桶的指标：“前端绑定”、“后端绑定”、“退休”、“错误推测”。比如说，我们发现程序执行的很大一部分被内存访问（这是一个“后端绑定”存储桶，见图@fig:TMA）停止了。下一步是再次运行工作负载并仅收集特定于“内存绑定”存储桶的指标（向下钻取）。重复该过程，直到我们知道确切的根本原因，例如，“L3 Bound”。


![The TMA hierarchy of performance bottlenecks. *© Image by Ahmad Yasin.*](/4/TMAM.png){#fig:TMA width=90%}

在实际应用中，性能可能会受到多种因素的限制。例如，它可以同时经历大量的分支错误预测（`Bad Speculation`）和缓存未命中（`Back End Bound`）。在这种情况下，TMA 将同时深入到多个存储桶中，并确定每种类型的瓶颈对程序性能的影响。 Intel VTune Profiler、AMD uprof 和 Linux `perf` 等分析工具可以通过一次基准测试计算所有指标。[^15]

TMA 指标的前两个级别以在程序执行期间可用的所有管道插槽的百分比表示（请参阅 [@sec:PipelineSlot]）。它允许 TMA 提供 CPU 微架构利用率的准确表示，同时考虑到处理器的全部带宽。

在我们确定程序中的性能瓶颈之后，我们会很想知道它发生在代码中的确切位置。 TMA 的第二阶段是将问题的根源定位到确切的代码行和汇编指令。分析方法提供了应该用于每一类性能问题的准确 PMC。然后，开发人员可以使用这个 PMC 在源代码中找到导致第一阶段确定的最关键性能瓶颈的区域。此对应关系可在“Locate-with”列中的 [TMA 指标](https://download.01.org/perfmon/TMA_Metrics.xlsx)[^2] 表中找到。例如，要定位在英特尔 Skylake 处理器上运行的应用程序中与高 `DRAM_Bound` 指标相关的瓶颈，应该对 `MEM_LOAD_RETIRED.L3_MISS_PS` 性能事件进行采样。

### 英特尔® VTune™ Profiler 中的 TMA

TMA 通过最新英特尔 VTune Profiler 中的“[微架构探索](https://software.intel.com/en-us/vtune-help-general-exploration-analysis)”[^3] 分析得到特色。图@fig:Vtune_GE 显示了 [7-zip benchmark](https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip) [^4] 的分析摘要。在图中，您可以看到由于 CPU 的“错误推测”，尤其是错误预测的分支，浪费了大量的执行时间。

![Intel VTune Profiler“微架构探索”分析。](/4/Vtune_GE.png){#fig:Vtune_GE width=90%}

该工具的美妙之处在于您可以单击您感兴趣的指标，该工具将带您进入显示对该特定指标有贡献的主要功能的页面。例如，如果您单击“Bad Speculation”指标，您将看到类似于图@fig:Vtune_GE_func 中所示的内容。 [^19]

![“微架构探索”自下而上视图。](/4/Vtune_GE_function_view.png){#fig:Vtune_GE_func width=90%}

从那里，如果您双击“LzmaDec_DecodeReal2”函数，英特尔® VTune™ Profiler 将带您进入源级视图，如图@fig:Vtune_GE_code 所示。突出显示的行导致 `LzmaDec_DecodeReal2` 函数中最大数量的分支错误预测。

!["Microarchitecture Exploration" source code and assembly view.](/4/Vtune_GE_code_view.png){#fig:Vtune_GE_code width=90%}

### TMA in Linux Perf {#sec:secTMA_perf}

As of Linux kernel 4.8, `perf` has an option `--topdown` used in `perf stat` command[^5] that prints TMA level 1 metrics, i.e., only four high-level buckets:

```bash
$ perf stat --topdown -a -- taskset -c 0 ./7zip-benchmark b
       retiring  bad speculat  FE bound  BE bound
S0-C0    30.8%      41.8%        8.8%      18.6%  <==
S0-C1    17.4%       2.3%       12.0%      68.2%
S0-C2    10.1%       5.8%       32.5%      51.6%
S0-C3    47.3%       0.3%        2.9%      49.6%
...
```

要获取高级 TMA 指标的值，Linux `perf` 需要分析整个系统（`-a`）。这就是为什么我们看到所有内核的指标。但是由于我们使用`taskset -c 0`将基准测试固定到core0，我们只能关注与`S0-C0`对应的第一行。

要访问 Top-Down 指标级别 2、3 等，可以使用 [toplev](https://github.com/andikleen/pmu-tools/wiki/toplev-manual)[^6] 工具，它是一种[pmu-tools](https://github.com/andikleen/pmu-tools)[^7] 的一部分，由 Andi Kleen 编写。它在“Python”中实现，并在底层调用 Linux 的“perf”。您将在下一节中看到使用它的示例。必须启用特定的 Linux 内核设置才能使用 `toplev`，查看文档了解更多详细信息。为了更好地展示工作流程，下一节将提供使用 TMA 提高内存受限应用程序性能的分步示例。

\personal{英特尔® VTune™ Profiler 是一个非常强大的工具，这一点毋庸置疑。然而，为了快速实验，我经常使用我正在开发的每个 Linux 发行版上都可用的 Linux 性能。因此，使用 Linux perf 探索下一节中示例的动机。}

### Step1：识别瓶颈

假设我们有一个运行 8.5 秒的小基准（`a.out`）。基准测试的完整源代码可以在 [github](https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM)[^8] 上找到。

```bash
$ time -p ./a.out
real 8.53
```

作为第一步，我们运行我们的应用程序并收集有助于我们表征它的特定指标，即我们尝试检测我们的应用程序属于哪个类别。 以下是我们基准测试的 1 级指标：[^9]

```bash
$ ~/pmu-tools/toplev.py --core S0-C0 -l1 -v --no-desc taskset -c 0 ./a.out
...
# Level 1
S0-C0  Frontend_Bound:   13.81 % Slots 
S0-C0  Bad_Speculation:   0.22 % Slots 
S0-C0  Backend_Bound:    53.43 % Slots <==
S0-C0  Retiring:         32.53 % Slots 
```
请注意，进程被固定到 CPU0（使用 `taskset -c 0`），并且 `toplev` 的输出仅限于该内核（`--core S0-C0`）。 通过查看输出，我们可以看出应用程序的性能受 CPU 后端的约束。 现在不尝试分析它，让我们向下钻取一层：[^16]

```bash
$ ~/pmu-tools/toplev.py --core S0-C0 -l2 -v --no-desc taskset -c 0 ./a.out
...
# Level 1
S0-C0  Frontend_Bound:                13.92 % Slots 
S0-C0  Bad_Speculation:                0.23 % Slots 
S0-C0  Backend_Bound:                 53.39 % Slots      
S0-C0  Retiring:                      32.49 % Slots   
# Level 2
S0-C0  Frontend_Bound.FE_Latency:     12.11 % Slots 
S0-C0  Frontend_Bound.FE_Bandwidth:    1.84 % Slots 
S0-C0  Bad_Speculation.Branch_Mispred: 0.22 % Slots 
S0-C0  Bad_Speculation.Machine_Clears: 0.01 % Slots 
S0-C0  Backend_Bound.Memory_Bound:    44.59 % Slots <==
S0-C0  Backend_Bound.Core_Bound:       8.80 % Slots 
S0-C0  Retiring.Base:                 24.83 % Slots 
S0-C0  Retiring.Microcode_Sequencer:   7.65 % Slots    
```
We see that the application’s performance is bound by memory accesses. Almost half of the CPU execution resources were wasted waiting for memory requests to complete. Now let us dig one level deeper: [^17]
```bash
$ ~/pmu-tools/toplev.py --core S0-C0 -l3 -v --no-desc taskset -c 0 ./a.out
...
# Level 1
S0-C0    Frontend_Bound:                 13.91 % Slots
S0-C0    Bad_Speculation:                 0.24 % Slots
S0-C0    Backend_Bound:                  53.36 % Slots
S0-C0    Retiring:                       32.41 % Slots
# Level 2
S0-C0    FE_Bound.FE_Latency:            12.10 % Slots
S0-C0    FE_Bound.FE_Bandwidth:           1.85 % Slots
S0-C0    BE_Bound.Memory_Bound:          44.58 % Slots
S0-C0    BE_Bound.Core_Bound:             8.78 % Slots
# Level 3
S0-C0-T0 BE_Bound.Mem_Bound.L1_Bound:     4.39 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.L2_Bound:     2.42 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.L3_Bound:     5.75 % Stalls
S0-C0-T0 BE_Bound.Mem_Bound.DRAM_Bound:  47.11 % Stalls <==
S0-C0-T0 BE_Bound.Mem_Bound.Store_Bound:  0.69 % Stalls
S0-C0-T0 BE_Bound.Core_Bound.Divider:     8.56 % Clocks
S0-C0-T0 BE_Bound.Core_Bound.Ports_Util: 11.31 % Clocks
```
我们发现瓶颈在“DRAM_Bound”中。 这告诉我们，许多内存访问在所有级别的缓存中都未命中，并一直向下到主内存。 如果我们收集程序的 L3 缓存未命中（DRAM 命中）的绝对数量，我们也可以确认这一点。 对于 Skylake 架构，“DRAM_Bound”指标是使用“CYCLE_ACTIVITY.STALLS_L3_MISS”性能事件计算的。 让我们收集它：

```bash
$ perf stat -e cycles,cycle_activity.stalls_l3_miss -- ./a.out
  32226253316  cycles
  19764641315  cycle_activity.stalls_l3_miss
```

根据 `CYCLE_ACTIVITY.STALLS_L3_MISS` 的定义，它在执行停止时计算周期，而 L3 缓存未命中需求负载未完成。 我们可以看到大约有 60% 的这样的周期，这是非常糟糕的。

### Step2: Locate the place in the code {#sec:secTMA_locate}

作为 TMA 过程的第二步，我们将定位代码中最常出现瓶颈的位置。为此，应使用与在步骤 1 中识别的瓶颈类型相对应的性能事件对工作负载进行采样。

查找此类事件的推荐方法是运行带有 `--show-sample` 选项的 `toplev` 工具，该选项将建议可用于定位问题的 `perf record` 命令行。为了理解 TMA 的机制，我们还提供了手动方法来查找与特定性能瓶颈相关的事件。可以借助 [TMA 指标](https://download.01.org/perfmon/TMA_Metrics.xlsx) 来完成性能瓶颈和性能事件之间的对应关系，该对应关系应用于定位代码中发生此类瓶颈的位置)[^2] 表在本章前面介绍过。 `Locate-with` 列表示一个性能事件，应该用于在代码中定位问题发生的确切位置。出于我们示例的目的，为了找到导致 DRAM_Bound 指标值如此高的内存访问（L3 缓存中未命中），我们应该对 MEM_LOAD_RETIRED.L3_MISS_PS 精确事件进行采样，如清单所示以下：

```bash
$ perf record -e cpu/event=0xd1,umask=0x20,name=MEM_LOAD_RETIRED.L3_MISS/ppp ./a.out

$ perf report -n --stdio
...
# Samples: 33K of event ‘MEM_LOAD_RETIRED.L3_MISS’
# Event count (approx.): 71363893
# Overhead   Samples  Shared Object      Symbol
# ........  ......... .................  .................
#
    99.95%    33811   a.out     [.] foo                
     0.03%       52   [kernel]  [k] get_page_from_freelist
     0.01%        3   [kernel]  [k] free_pages_prepare
     0.00%        1   [kernel]  [k] free_pcppages_bulk
```
大多数 L3 未命中是由可执行文件 `a.out` 中的函数 `foo` 中的内存访问引起的。 为了避免编译器优化，函数 `foo` 用汇编语言实现，在 [@lst:TMA_asm] 中介绍。 基准的“驱动程序”部分在 `main` 函数中实现，如 [@lst:TMA_cpp] 所示。 我们分配了一个足够大的数组 `a` 以使其不适合 L3 缓存[^10]。 基准测试基本上为数组“a”生成一个随机索引，并将它与数组“a”的地址一起传递给“foo”函数。 稍后 `foo` 函数会读取这个随机内存位置。 [^11]

清单：函数 foo 的汇编代码。

~~~~ {#lst:TMA_asm .bash}
$ perf annotate --stdio -M intel foo
Percent |  Disassembly of a.out for MEM_LOAD_RETIRED.L3_MISS
------------------------------------------------------------
        :  Disassembly of section .text:
        :
        :  0000000000400a00 <foo>:
        :  foo():
   0.00 :    400a00:  nop  DWORD PTR [rax+rax*1+0x0]
   0.00 :    400a08:  nop  DWORD PTR [rax+rax*1+0x0]
                 ...
 100.00 :    400e07:  mov  rax,QWORD PTR [rdi+rsi*1] <==
                 ...
   0.00 :    400e13:  xor  rax,rax
   0.00 :    400e16:  ret 
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Listing: Source code of function main.

~~~~ {#lst:TMA_cpp .cpp}
extern "C" { void foo(char* a, int n); }
const int _200MB = 1024*1024*200;
int main() {
  char* a = (char*)malloc(_200MB); // 200 MB buffer
  ...
  for (int i = 0; i < 100000000; i++) {
    int random_int = distribution(generator);
    foo(a, random_int);
  }
  ...
}
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

通过查看 [@lst:TMA_asm]，我们可以看到函数 `foo` 中的所有 L3-Cache 未命中都标记为单个指令。 现在我们知道哪条指令导致了这么多 L3 未命中，让我们修复它。

### 第三步：修复问题

因为在我们获得下一个将被访问的地址和实际加载指令之间有一个时间窗口，我们可以添加一个预取提示[^12]，如 [@lst:TMA_prefetch] 所示。 有关内存预取的更多信息，请参见 [@sec:memPrefetch]。

Listing: Inserting memory prefetch into main.

~~~~ {#lst:TMA_prefetch .cpp}
  for (int i = 0; i < 100000000; i++) {
    int random_int = distribution(generator);
+   __builtin_prefetch ( a + random_int, 0, 1);
    foo(a, random_int);
  }
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This hint improved execution time by 2 seconds, which is a 30% speedup. Notice 10x less value for `CYCLE_ACTIVITY.STALLS_L3_MISS` event:

```bash
$ perf stat -e cycles,cycle_activity.stalls_l3_miss -- ./a.out
  24621931288      cycles
   2069238765      cycle_activity.stalls_l3_miss
       6,498080824 seconds time elapsed
```

TMA 是一个迭代过程，因此我们现在需要从 Step1 开始重复该过程。很可能它会将瓶颈转移到其他存储桶中，在这种情况下，就是 Retiring。这是一个简单的例子，展示了 TMA 方法的工作流程。分析现实世界的应用程序不太可能那么容易。本书下一整章的组织方式便于与 TMA 流程一起使用。例如，它的部分被分解以反映每个高级别的性能瓶颈类别。这种结构背后的想法是提供某种清单，开发人员可以在发现性能问题后使用该清单来驱动代码更改。例如，当开发人员看到他们正在开发的应用程序是“内存绑定”时，他们可以查找 [@sec:MemBound] 的想法。

＃＃＃ 概括

TMA 非常适合识别代码中的 CPU 性能瓶颈。理想情况下，当我们在某个应用程序上运行它时，我们希望看到 Retiring 指标为 100%。这意味着该应用程序完全使 CPU 饱和。在玩具程序上可以实现接近此的结果。然而，现实世界的应用程序远未实现。图@fig:TMA_metrics_SPEC2006 显示了 Skylake CPU 生成的 [SPEC CPU2006](http://spec.org/cpu2006/)[^13] 基准测试的顶级 TMA 指标。请记住，随着架构师不断尝试改进 CPU 设计，其他代 CPU 的数字可能会发生变化。对于其他指令集架构 (ISA) 和编译器版本，这些数字也可能会发生变化。

![SPEC CPU2006 的顶级 TMA 指标。 *© 图片由 Ahmad Yasin 提供，[http://cs.haifa.ac.il/~yosi/PARC/yasin.pdf](http://cs.haifa.ac.il/~yosi/PARC/yasin.pdf ).*](/4/TMAM_metrics_SPEC2006.png){#fig:TMA_metrics_SPEC2006 width=90%}

不建议在具有重大性能缺陷的代码上使用 TMA，因为它可能会将您引向错误的方向，并且您将调整错误的代码，而不是修复真正的高级性能问题，这只是浪费时间。同样，确保环境不会妨碍分析。例如，如果您删除文件系统缓存并在 TMA 下运行基准测试，它可能会显示您的应用程序受内存限制，事实上，当文件系统缓存预热时，这可能是错误的。

TMA 提供的工作负载特征可以扩大源代码之外的潜在优化范围。例如，如果应用程序受内存限制，并且检查了在软件级别上加速它的所有可能方法，则可以通过使用更快的内存来改进内存子系统。这可以进行有根据的实验，因为只有在您发现程序受内存限制并且它将受益于更快的内存时，才会花掉这笔钱。

在撰写本文时，第一级 TMA 指标也可用于 AMD 处理器。

**其他资源和链接：**

* Ahmad Yasin 的论文“A top-down method for performance analysis and counters architecture” [@TMA_ISPASS]。

- Presentation "Software Optimizations Become Simple with Top-Down Analysis on Intel Skylake" by Ahmad Yasin at IDF'15, URL: [https://youtu.be/kjufVhyuV_A](https://youtu.be/kjufVhyuV_A).
- Andi Kleen's blog - pmu-tools, part II: toplev, URL: [http://halobates.de/blog/p/262](http://halobates.de/blog/p/262).
- Toplev manual, URL: [https://github.com/andikleen/pmu-tools/wiki/toplev-manual](https://github.com/andikleen/pmu-tools/wiki/toplev-manual).
- Understanding How General Exploration Works in Intel® VTune™ Profiler, URL: [https://software.intel.com/en-us/articles/understanding-how-general-exploration-works-in-intel-vtune-amplifier-xe](https://software.intel.com/en-us/articles/understanding-how-general-exploration-works-in-intel-vtune-amplifier-xe).

[^2]: TMA metrics - [https://download.01.org/perfmon/TMA_Metrics.xlsx](https://download.01.org/perfmon/TMA_Metrics.xlsx).
[^3]: VTune microarchitecture analysis - [https://software.intel.com/en-us/vtune-help-general-exploration-analysis](https://software.intel.com/en-us/vtune-help-general-exploration-analysis). In pre-2019 versions of Intel® VTune Profiler, it was called as “General Exploration” analysis.
[^4]: 7zip benchmark - [https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip](https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip).
[^5]: Linux `perf stat` manual page - [http://man7.org/linux/man-pages/man1/perf-stat.1.html#STAT_REPORT](http://man7.org/linux/man-pages/man1/perf-stat.1.html#STAT_REPORT).
[^6]: Toplev - [https://github.com/andikleen/pmu-tools/wiki/toplev-manual](https://github.com/andikleen/pmu-tools/wiki/toplev-manual)
[^7]: PMU tools - [https://github.com/andikleen/pmu-tools](https://github.com/andikleen/pmu-tools).
[^8]: Benchmark for TMA section - [https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM](https://github.com/dendibakh/dendibakh.github.io/tree/master/_posts/code/TMAM).
[^9]: Outputs in this section are trimmed to fit on the page. Do not rely on the exact format that is presented.
[^10]: L3 cache on the machine I was using is 38.5 MB - Intel(R) Xeon(R) Platinum 8180 CPU.
[^11]: According to x86 calling conventions ([https://en.wikipedia.org/wiki/X86_calling_conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)), first 2 arguments land in `rdi` and `rsi` registers respectively.
[^12]:  Documentation about `__builtin_prefetch` can be found at [https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html).
[^13]: SPCE CPU 2006 - [http://spec.org/cpu2006/](http://spec.org/cpu2006/).
[^14]: In reality, it is sufficient to run the workload once to collect all the metrics required for TMA. Profiling tools achieve that by multiplexing between different performance events during a single run (see [@sec:secMultiplex]).
[^15]: This only is acceptable if the workload is steady. Otherwise, you would better fall back to the original strategy of multiple runs and drilling down with each run.
[^16]: Alternatively, we could use `-l1 --nodes Core_Bound,Memory_Bound` instead of `-l2` to limit the collection of all the metrics since we know from the first level metrics that the application is bound by CPU Backend.
[^17]: Alternatively, we could use `-l2 --nodes L1_Bound,L2_Bound,L3_Bound,DRAM_Bound,Store_Bound,` `Divider,Ports_Utilization` option instead of `-l3` to limit collection, since we knew from the second level metrics that the application is bound by memory.
[^19]: Per-function view of TMA metrics is a feature unique to Intel® VTune profiler.
