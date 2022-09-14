---
typora-root-url: ..\..\img
---

## Pipeline Slot {#sec:PipelineSlot}

流水线槽代表处理一个微指令所需的硬件资源。 图@fig:PipelineSlot 演示了一个 CPU 的执行管道，每个周期可以处理 4 个微指令。 几乎所有现代 x86 CPU 的流水线宽度都是 4（4 宽）。 在图表上的六个连续周期中，只有一半的可用时隙被使用。 从微架构的角度来看，执行此类代码的效率仅为 50%。

![Pipeline diagram of a 4-wide CPU.](/3/PipelineSlot.jpg){#fig:PipelineSlot width=40% }

管道插槽是自上而下微架构分析中的核心指标之一（参见 [@sec:TMA]）。 例如，由于各种原因，前端绑定和后端绑定指标表示为未使用管道插槽的百分比。
