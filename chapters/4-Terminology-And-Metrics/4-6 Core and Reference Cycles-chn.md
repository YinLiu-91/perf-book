---
typora-root-url: ..\..\img
---

## Core vs. Reference Cycles {#sec:secRefCycles}

大多数 CPU 使用时钟信号来调整它们的顺序操作。时钟信号由每秒提供一致数量的脉冲的外部发生器产生。时钟脉冲的频率决定了 CPU 执行指令的速率。因此，时钟越快，CPU 每秒执行的指令就越多。
$$
频率 = \frac{时钟滴答声}{时间}
$$
大多数现代 CPU，包括 Intel 和 AMD CPU，都没有固定的运行频率。相反，它们实现了[动态频率缩放](https://en.wikipedia.org/wiki/Dynamic_frequency_scaling)[^5]。在 Intel 的 CPU 中，这项技术称为 [Turbo Boost](https://en.wikipedia.org/wiki/Intel_Turbo_Boost)[^6]，在 AMD 的处理器中称为 [Turbo Core](https://en.wikipedia.org /wiki/AMD_Turbo_Core)[^7]。它允许 CPU 动态地增加和降低其频率 - 调整频率会以牺牲性能为代价来降低功耗，而提高频率会提高性能但会牺牲节能。

内核时钟周期计数器以 CPU 内核运行的实际时钟频率计算时钟周期，而不是外部时钟（参考周期）。我们来看一个在 Skylake i7-6000 处理器上的实验，它的基频为 3.4 GHz：

```bash
$ perf stat -e cycles,ref-cycles ./a.exe
  43340884632  cycles		# 3.97 GHz
  37028245322  ref-cycles	# 3.39 GHz
      10,899462364 seconds time elapsed
```

指标 `ref-cycles` 对周期进行计数，就好像没有频率缩放一样。 设置上的外部时钟频率为 100 MHz，如果我们通过 [时钟乘数](https://en.wikipedia.org/wiki/CPU_multiplier) 对其进行缩放，我们将获得处理器的基本频率。 Skylake i7-6000 处理器的时钟倍数等于 34：这意味着对于每个外部脉冲，CPU 在基本频率上运行时执行 34 个内部周期。

指标“周期”计算实际 CPU 周期，即考虑频率缩放。 我们还可以计算出动态频率缩放功能的使用情况：
$$
Turbo~Utilization = \frac{Core~Cycles}{Reference~Cycles},
$$

核心时钟周期计数器在测试哪个版本的代码最快时非常有用，因为您可以避免时钟频率上升和下降的问题。[@fogOptimizeCpp]

[^5]: Dynamic frequency scaling - [https://en.wikipedia.org/wiki/Dynamic_frequency_scaling](https://en.wikipedia.org/wiki/Dynamic_frequency_scaling).
[^6]: Intel Turbo Boost - [https://en.wikipedia.org/wiki/Intel_Turbo_Boost](https://en.wikipedia.org/wiki/Intel_Turbo_Boost).
[^7]: AMD Turbo Core - [https://en.wikipedia.org/wiki/AMD_Turbo_Core](https://en.wikipedia.org/wiki/AMD_Turbo_Core).
