---
typora-root-url: ..\..\img
---

# Part1. Performance analysis on a modern CPU {.unnumbered}

# Measuring Performance {#sec:secMeasPerf}

了解应用程序性能的第一步是知道如何衡量它。有些人将性能归因于应用程序的特性之一[^15]。但与其他功能不同的是，性能不是布尔属性：应用程序总是具有某种程度的性能。这就是为什么对于应用程序是否具有性能的问题无法回答“是”或“否”的原因。

性能问题通常比大多数功能问题更难追踪和重现[^10]。基准测试的每次运行都彼此不同。例如，当解压一个 zip 文件时，我们一遍又一遍地得到相同的结果，这意味着这个操作是可重现的 [^16]。但是，不可能重现此操作的完全相同的性能配置文件。

任何关心绩效评估的人都可能知道进行公平的绩效衡量并从中得出准确的结论是多么困难。性能测量有时可能非常出乎意料。更改源代码中看似无关的部分可能会对程序性能产生重大影响，让我们感到惊讶。这种现象称为测量偏差。由于测量中存在误差，性能分析需要统计方法来处理它们。这个话题本身就值得一整本书。在这个领域有很多极端案例和大量研究。我们不会一路走下这个兔子洞。相反，我们将只关注高层次的想法和​​要遵循的方向。

进行公平的性能实验是获得准确和有意义的结果的重要一步。设计性能测试和配置环境都是评估性能过程中的重要组成部分。本章将简要介绍现代系统产生噪声性能测量的原因以及您可以做些什么。我们将谈到在实际生产部署中衡量性能的重要性。

没有一个长期存在的产品没有性能回归。这对于具有大量贡献者且变化速度非常快的大型项目尤其重要。本章用几页篇幅讨论在持续集成和持续交付 (CI/CD) 系统中跟踪性能变化的自动化过程。我们还提供了有关在开发人员在其源代码库中实施更改时如何正确收集和分析性能测量的一般指导。

本章末尾描述了开发人员可以在基于时间的测量中使用的软件和硬件定时器，以及在设计和编写良好的微基准测试时常见的陷阱。

[^10]: Sometimes, we have to deal with non-deterministic and hard to reproduce bugs, but it's not that often.
[^15]: Blog post by Nelson Elhage "Reflections on software performance": [https://blog.nelhage.com/post/reflections-on-performance/](https://blog.nelhage.com/post/reflections-on-performance/).
[^16]: Assuming no data races.
