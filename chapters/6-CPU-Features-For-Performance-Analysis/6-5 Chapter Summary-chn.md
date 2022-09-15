---
typora-root-url: ..\..\img
---

## Chapter Summary

* 仅在修复所有高级性能问题后，才建议使用硬件功能进行低级调优。调整设计不佳的算法是对开发人员时间的不良投资。一旦消除了所有主要的性能问题，就可以使用 CPU 性能监控功能来分析和进一步调整他们的应用程序。
* 自上而下的微架构分析 (TMA) 方法是一种非常强大的技术，用于识别程序对 CPU 微架构的无效使用。它是一种强大而正式的方法，即使对于没有经验的开发人员也很容易使用。 TMA 是一个迭代过程，由多个步骤组成，包括表征工作负载和定位源代码中出现瓶颈的确切位置。我们建议 TMA 应该是每个低级调优工作的分析起点。 TMA 在 Intel 和 AMD[^1] 处理器上可用。
* Last Branch Record (LBR) 机制在执行程序的同时连续记录最近的分支结果，从而将速度减慢。它允许我们为我们在分析时收集的每个样本拥有足够深的调用堆栈。此外，LBR 有助于识别热分支、误预测率并允许对机器代码进行精确计时。 Intel 和 AMD 处理器支持 LBR。
* 基于处理器事件的采样 (PEBS) 功能是分析的另一项增强功能。它通过在没有中断的情况下自动多次采样到专用缓冲区来降低采样开销。然而，PEBS 更广为人知的是引入了“精确事件”，它允许精确定位导致特定性能事件的精确指令。 Intel 处理器支持该功能。 AMD CPU 具有类似的功能，称为基于指令的采样 (IBS)。
* 英特尔处理器跟踪 (PT) 是一项 CPU 功能，它通过以高度压缩的二进制格式对数据包进行编码来记录程序执行情况，该格式可用于重建每个指令上带有时间戳的执行流程。 PT 具有广泛的覆盖范围和相对较小的开销。它的主要用途是事后分析和查找性能故障的根本原因。基于 ARM 架构的处理器还具有称为 [CoreSight](https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace)[^2] 的跟踪能力，但主要是用于调试而不是性能分析。

性能分析器利用本章介绍的硬件特性来支持许多不同类型的分析。

[^1]: Although at the time of writing, AMD processors only support the first level of TMA metrics, i.e., Front End Bound, Back End Bound, Retiring, and Bad Speculation.
[^2]: ARM CoreSight: [https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace](https://developer.arm.com/ip-products/system-ip/coresight-debug-and-trace)

\sectionbreak

