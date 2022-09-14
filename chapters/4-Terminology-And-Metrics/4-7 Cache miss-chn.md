---
typora-root-url: ..\..\img
---

## Cache miss

正如 [@sec:MemHierar] 中所讨论的，在特定级别的缓存中丢失的任何内存请求都必须由更高级别的缓存或 DRAM 提供服务。 这意味着这种内存访问的延迟显着增加。 内存子系统组件的典型延迟如表 {@tbl:mem_latency} 所示。[^1] 性能受到很大影响，尤其是当内存请求在最后一级缓存 (LLC) 中丢失并一直向下传输到主内存时 ( 动态随机存取存储器）。 英特尔® [内存延迟检查器](https://www.intel.com/software/mlc)[^2] (MLC) 是一种用于测量内存延迟和带宽以及它们如何随着系统负载增加而变化的工具。 MLC 可用于为被测系统建立基线和进行性能分析。

-------------------------------------------------
Memory Hierarchy Component   Latency (cycle/time)

--------------------------   --------------------
L1 Cache                     4 cycles (~1 ns)

L2 Cache                     10-25 cycles (5-10 ns)

L3 Cache                     ~40 cycles (20 ns)

Main                         200+ cycles (100 ns)
Memory

-------------------------------------------------

表：内存子系统的典型延迟。 {#tbl:mem_latency}

指令和数据都可能发生缓存未命中。 根据自上而下的微架构分析（参见 [@sec:TMA]），指令（I-cache）缓存未命中被表征为 Front-End Stall，而数据缓存（D-cache）未命中被表征为 Back -结束摊位。 在指令获取期间发生 I-cache 未命中时，它被归为前端问题。 因此，当此加载请求的数据在 D-cache 中未找到时，这将被归类为后端问题。

Linux `perf` 用户可以通过运行以下命令收集 L1-cache 未命中数：

```bash
$ perf stat -e mem_load_retired.fb_hit,mem_load_retired.l1_miss,
  mem_load_retired.l1_hit,mem_inst_retired.all_loads -- a.exe
   29580  mem_load_retired.fb_hit
   19036  mem_load_retired.l1_miss
  497204  mem_load_retired.l1_hit
  546230  mem_inst_retired.all_loads
```

以上是 L1 数据缓存的所有负载的细分。 我们可以看到 L1 缓存中只有 3.5% 的负载未命中。 我们可以通过运行进一步分解 L1 数据未命中并分析 L2 缓存行为：

```bash
$ perf stat -e mem_load_retired.l1_miss,
  mem_load_retired.l2_hit,mem_load_retired.l2_miss -- a.exe
  19521  mem_load_retired.l1_miss
  12360  mem_load_retired.l2_hit
   7188  mem_load_retired.l2_miss
```
从这个示例中，我们可以看到在 L1 D-cache 中丢失的负载中有 37% 也在 L2 缓存中丢失。 以类似的方式，可以对 L3 缓存进行细分。


[^1]: There is also an interactive view that visualizes the cost of different operations in modern systems: [https://colin-scott.github.io/personal_website/research/interactive_latency.html](https://colin-scott.github.io/personal_website/research/interactive_latency.html).
[^2]: Memory Latency Checker - [https://www.intel.com/software/mlc](https://www.intel.com/software/mlc).
