---
typora-root-url: ..\..\img
---

## Workload Characterization {#sec:counting}

工作负载表征是通过定量参数和函数来描述工作负载的过程。 它的目标是定义工作负载的行为及其最重要的特性。 在高层次上，应用程序可以属于以下一种或多种类型：交互式、数据库、基于网络、并行等。可以使用不同的指标和参数来表征不同的工作负载，以解决特定的应用程序域。

在 [@sec:TMA] 中，我们将仔细研究自上而下的微架构分析 (TMA) 方法，该方法试图通过将应用程序放入 4 个存储桶之一来表征应用程序：前端绑定、后端绑定、退休和坏 猜测。 TMA 使用性能监控计数器（PMC，参见 [@sec:PMC]）来收集所需信息并识别 CPU 微架构的低效使用。

### Counting Performance Events

PMC 是一种非常重要的低级性能分析工具。 他们可以提供有关我们程序执行的独特信息。 PMC 通常以两种模式使用：“计数”和“采样”。 计数模式用于工作负载表征，而采样模式用于查找热点，我们将在 [@sec:profiling] 中讨论。 Counting 背后的想法非常简单：我们想计算程序运行期间某些性能事件的数量。 图@fig:Counting 说明了从时间角度对性能事件进行计数的过程。

![Counting performance events.](/2/CountingFlow.png){#fig:Counting width=90%}

图@fig:Counting 中概述的步骤大致代表了典型分析工具对性能事件的计数。 此过程在 `perf stat` 工具中实现，可用于统计各种硬件事件，如指令数、周期数、缓存未命中等。以下是 `perf stat` 的输出示例：

```bash
$ perf stat -- ./a.exe
 10580290629  cycles         #    3,677 GHz
  8067576938  instructions   #    0,76  insn per cycle
  3005772086  branches       # 1044,472 M/sec
   239298395  branch-misses  #    7,96% of all branches 
```

了解这些数据非常有用。首先，它使我们能够快速发现一些异常情况，例如高速缓存未命中率或低 IPC。但是，当您刚刚进行代码改进并且想要验证性能提升时，它可能会派上用场。查看绝对数字可能会帮助您证明或拒绝代码更改。

\personal{我使用 `perf stat` 作为简单的基准测试包装器。由于计数事件的开销很小，我在“perf stat”下自动运行几乎所有的基准测试。它是我进行绩效调查的第一步。有时可以立即发现异常，从而节省一些分析时间。}

### 手动性能计数器收集

现代 CPU 有数百个可数的性能事件。很难记住所有这些及其含义。了解何时使用特定 PMC 更加困难。这就是为什么通常我们不建议手动收集特定的 PMC，除非您真的知道自己在做什么。相反，我们建议使用 Intel Vtune Profiler 等工具来自动执行此过程。然而，在某些情况下，您有兴趣收集特定的 PMC。

可以在 [@IntelSDM，第 3B 卷，第 19 章] 中找到所有 Intel CPU 代的性能事件的完整列表。[^1] 每个事件都使用 `Event` 和 `Umask` 十六进制值进行编码。有时，性能事件也可以使用附加参数进行编码，例如“Cmask”和“Inv”等。表 {@tbl:perf_count} 中显示了为英特尔 Skylake 微架构编码两个性能事件的示例。

--------------------------------------------------------------------------
Event  Umask Event Mask            Description
 Num.  Value Mnemonic              
------ ----- --------------------- ---------------------------------------
C0H     00H  INST_RETIRED.         Number of instructions at retirement. 
             ANY_P

C4H     00H  BR_INST_RETIRED.      Branch instructions that retired.
             ALL_BRANCHES                  
--------------------------------------------------------------------------

表：编码 Skylake 性能事件的示例。 {#tbl:perf_count}

Linux `perf` 提供常用性能计数器的映射。 它们可以通过伪名称访问，而不是指定 `Event` 和 `Umask` 十六进制值。 例如，`branches` 只是 `BR_INST_RETIRED.ALL_BRANCHES` 的同义词，将测量相同的东西。 可以使用 `perf list` 查看可用映射名称的列表：

```bash
$ perf list
  branches          [Hardware event]
  branch-misses     [Hardware event]
  bus-cycles        [Hardware event]
  cache-misses      [Hardware event]
  cycles            [Hardware event]
  instructions      [Hardware event]
  ref-cycles        [Hardware event]
```

但是，Linux `perf` 并没有为每个 CPU 架构的所有性能计数器提供映射。 如果您要查找的 PMC 没有映射，则可以使用以下语法收集它：

```重击
$ perf stat -e cpu/event=0xc4,umask=0x0,name=BR_INST_RETIRED.ALL_BRANCHES/ -- ./a.exe
```

还有一些围绕 Linux `perf` 的包装器可以进行映射，例如 [oprofile](https://oprofile.sourceforge.io/about/)[^2] 和 [ocperf.py](https:// github.com/andikleen/pmu-tools/blob/master/ocperf.py)[^3]。 以下是它们的用法示例：

```bash
$ ocperf -e uops_retired ./a.exe
$ ocperf.py stat -e uops_retired.retire_slots -- ./a.exe
```

性能计数器并非在每个环境中都可用，因为访问 PMC 需要 root 访问权限，而在虚拟化环境中运行的应用程序通常没有这些权限。对于在公共云中执行的程序，如果虚拟机 (VM) 管理器未将 PMU 编程接口正确地暴露给来宾，则直接在来宾容器中运行基于 PMU 的分析器不会产生有用的输出。因此，基于 CPU 性能计数器的分析器在虚拟化和云环境中无法正常工作 [@PMC_virtual]。虽然情况正在好转。 VmWare® 是首批启用虚拟 CPU 性能计数器 (vPMC) 的 VM 管理器之一。 [^4] 为专用主机启用 AWS EC2 云的 PMC。 [^5]

### 多路复用和缩放事件 {#sec:secMultiplex}

在某些情况下，我们想同时计算许多不同的事件。但是只有一个计数器，一次只能计算一件事。这就是 PMU 中有多个计数器的原因（通常每个硬件线程 4 个）。即使这样，固定和可编程计数器的数量也并不总是足够的。自上而下的分析方法 (TMA) 要求在一次程序执行中收集多达 100 个不同的性能事件。显然，CPU 没有那么多计数器，这就是多路复用发挥作用的时候。

如果事件多于计数器，则分析工具使用时间多路复用来为每个事件提供访问监控硬件的机会。图@fig:Multiplexing 显示了一个在 8 个性能事件之间进行多路复用的示例，其中只有 4 个 PMC 可用。

<div id="fig:复用">
![](/2/Multiplexing1.png){#fig:Multiplexing1 宽度=60%}

![](/2/Multiplexing2.png){#fig:Multiplexing2 宽度=60%}

在 8 个性能事件之间多路复用，只有 4 个 PMC 可用。
</div>

使用多路复用，事件不会一直被测量，而只是在它的某个部分期间测量。在运行结束时，分析工具需要根据启用的总时间来缩放原始计数：
$$
final~count = raw~count * ( time~running / time~enabled )
$$
例如，在分析期间，我们能够在五个时间间隔内测量一些计数器。每个测量间隔持续 100 毫秒（“时间启用”）。程序运行时间为1s（`time running`）。此计数器的事件总数测量为 10000（“原始计数”）。因此，我们需要将“最终计数”缩放 2，这将等于 20000：
$$
最终~计数 = 10000 * ( 1000ms / 500ms ) = 20000
$$
这提供了一个估计，如果在整个运行期间测量事件，计数将是多少。了解这是一个估计值，而不是实际计数非常重要。多路复用和缩放可以安全地用于在长时间间隔内执行相同代码的稳定工作负载。相反，如果程序有规律地在不同的热点之间跳转，就会出现盲点，从而在缩放过程中引入错误。为了避免扩展，可以尝试将事件的数量减少到不大于可用的物理 PMC 的数量。但是，这将需要多次运行基准测试来测量所有感兴趣的计数器。

[^1]: PMCs description is also available here: [https://download.01.org/perfmon/index/](https://download.01.org/perfmon/index/).
[^2]: Oprofile - [https://oprofile.sourceforge.io/about/](https://oprofile.sourceforge.io/about/)
[^3]: PMU tools - [https://github.com/andikleen/pmu-tools/blob/master/ocperf.py](https://github.com/andikleen/pmu-tools/blob/master/ocperf.py)
[^4]: VMWare PMCs - [https://www.vladan.fr/what-are-vmware-virtual-cpu-performance-monitoring-counters-vpmcs/](https://www.vladan.fr/what-are-vmware-virtual-cpu-performance-monitoring-counters-vpmcs/)
[^5]: Amazon EC2 PMCs - [http://www.brendangregg.com/blog/2017-05-04/the-pmcs-of-ec2.html](http://www.brendangregg.com/blog/2017-05-04/the-pmcs-of-ec2.html)
