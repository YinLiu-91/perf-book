---
typora-root-url: ..\..\img
---

## Static Performance Analysis

今天，我们拥有广泛的静态代码分析工具。对于 C 和 C++ 语言，我们有一些众所周知的工具，例如 [Clang 静态分析器](https://clang-analyzer.llvm.org/)、[Klocwork](https://www.perforce.com/products/klocwork) , [Cppcheck](http://cppcheck.sourceforge.net/) 和其他[^1]。它们旨在检查代码的正确性和语义。同样，也有一些工具试图解决代码的性能问题。静态性能分析器不运行实际代码。相反，他们模拟代码，就好像它在真正的硬件上执行一样。静态预测性能几乎是不可能的，因此这种类型的分析有很多限制。

首先，不可能静态分析 C/C++ 代码的性能，因为我们不知道它将被编译成的机器代码。因此，静态性能分析适用于汇编代码。

其次，静态分析工具模拟工作负载而不是执行它。它显然很慢，因此不可能静态分析整个程序。相反，工具采用一些汇编代码片段并尝试预测它在真实硬件上的行为。用户应该选择特定的汇编指令（通常是小循环）进行分析。所以，静态性能分析的范围很窄。

静态分析器的输出相当低级，有时会将执行分解为 CPU 周期。通常，开发人员使用它来对每个周期都很重要的关键代码区域进行细粒度调整。

### 静态与动态分析器

**静态工具**不运行实际代码，而是尝试模拟执行，尽可能多地保留微架构细节。它们无法进行真正的测量（执行时间、性能计数器），因为它们不运行代码。这里的好处是您不需要拥有真正的硬件，并且可以模拟不同 CPU 代的代码。另一个好处是您无需担心结果的一致性：静态分析器将始终为您提供稳定的输出，因为模拟（与真实硬件上的执行相比）没有任何偏差。静态工具的缺点是它们通常无法预测和模拟现代 CPU 中的所有内容：它们基于某些可能存在错误和限制的模型。静态性能分析器的示例有 [IACA](https://software.intel.com/en-us/articles/intel-architecture-code-analyzer)[^2] 和 [llvm-mca](https://llvm .org/docs/CommandGuide/llvm-mca.html)[^3]。

**动态工具**基于在真实硬件上运行代码并收集有关执行的各种信息。这是证明任何性能假设的唯一 100% 可靠方法。不利的一面是，您通常需要拥有特权访问权限才能收集 PMC 等低级性能数据。编写一个好的基准并衡量您想要衡量的内容并不总是那么容易。最后，您需要过滤噪音和各种副作用。动态性能分析器的示例有 Linux perf、[likwid](https://github.com/RRZE-HPC/likwid)[^5] 和 [uarch-bench](https://github.com/travisdowns/uarch-长凳）[^4]。上述工具的使用和输出示例可以在 [easyperf 博客](https://easyperf.net/blog/2018/04/03/Tools-for-microarchitectural-benchmarking) [^6] 上找到。

[这里](https://github.com/MattPD/cpplinks/blob/master/performance.tools.md#microarchitecture)[^7] 提供了大量用于静态和动态微架构性能分析的工具。

\personal{每当我需要探索一些有趣的 CPU 微架构效果时，我都会使用这些工具。静态和低级动态分析器（如 likwid 和 uarch-bench）允许我们在进行性能实验时在实践中观察硬件效应。它们对于建立 CPU 工作原理的心智模型有很大帮助。}

[^1]: Tools for static code analysis - [https://en.wikipedia.org/wiki/List_of_tools_for_static_code_analysis#C,_C++](https://en.wikipedia.org/wiki/List_of_tools_for_static_code_analysis#C,_C++).
[^2]: IACA - [https://software.intel.com/en-us/articles/intel-architecture-code-analyzer](https://software.intel.com/en-us/articles/intel-architecture-code-analyzer). In April 2019, the tools has reached its End Of Life and is no longer supported.
[^3]: LLVM MCA - [https://llvm.org/docs/CommandGuide/llvm-mca.html](https://llvm.org/docs/CommandGuide/llvm-mca.html)
[^4]: Uarch bench - [https://github.com/travisdowns/uarch-bench](https://github.com/travisdowns/uarch-bench)
[^5]: LIKWID - [https://github.com/RRZE-HPC/likwid](https://github.com/RRZE-HPC/likwid)
[^6]: An article about tools for microarchitectural benchmarking - [https://easyperf.net/blog/2018/04/03/Tools-for-microarchitectural-benchmarking](https://easyperf.net/blog/2018/04/03/Tools-for-microarchitectural-benchmarking)
[^7]: Collection of links for C++ performance tools - [https://github.com/MattPD/cpplinks/blob/master/performance.tools.md#microarchitecture](https://github.com/MattPD/cpplinks/blob/master/performance.tools.md#microarchitecture).
