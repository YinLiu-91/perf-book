---
typora-root-url: ..\..\img
---

## What is discussed in this book?


本书旨在帮助开发人员更好地了解他们的应用程序的性能，学会发现低效率并消除它们。 *为什么我的手写存档器的执行速度比传统存档器慢两倍？为什么我的功能改变导致两倍的性能下降？客户抱怨我的应用程序运行缓慢，而我不知道从哪里开始？我是否优化了程序以充分发挥其潜力？我该如何处理所有缓存未命中和分支错误预测？* 希望在本书结束时，您将获得这些问题的答案。

以下是本书内容的概要：

* 第 2 章讨论如何进行公平的性能实验并分析其结果。它介绍了性能测试和比较结果的最佳实践。
* 第 3 章和第 4 章提供 CPU 微架构基础知识和性能分析术语；如果您已经知道这一点，请随时跳过。
* 第 5 章探讨了几种最流行的性能分析方法。它解释了分析技术如何工作以及可以收集哪些数据。
* 第 6 章介绍了现代 CPU 为支持和增强性能分析而提供的特性信息。它显示了它们的工作方式以及它们能够解决的问题。
* 第 7-9 章包含典型性能问题的方法。它以最方便的方式与自顶向下微体系结构分析一起使用（参见 [@sec:TMA]），这是本书最重要的概念之一。
* 第 10 章包含的优化主题与前三章中涵盖的任何类别都没有特别相关，但仍然很重要，足以在本书中找到它们的位置。
* 第 11 章讨论了分析多线程应用程序的技术。它概述了优化多线程应用程序性能的一些最重要的挑战以及可用于分析它的工具。这个话题本身就比较大，所以本章只关注硬件特定的问题，比如“虚假共享”。

本书中提供的示例主要基于开源软件：Linux 作为操作系统，基于 LLVM 的 Clang 编译器用于 C 和 C++ 语言，Linux `perf` 作为分析工具。做出这样选择的原因不仅是上述技术的普及，而且它们的源代码是开放的，这使我们能够更好地了解它们工作的底层机制。这对于学习本书中介绍的概念特别有用。我们有时还会展示在其领域“大玩家”的专有工具，例如英特尔® VTune™ Profiler。