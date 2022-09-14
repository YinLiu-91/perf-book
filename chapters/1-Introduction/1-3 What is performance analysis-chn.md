---
typora-root-url: ..\..\img
---

## What Is Performance Analysis?

有没有发现自己与同事争论某段代码的性能？那么您可能知道预测哪些代码会运行得最好是多么困难。现代处理器中有如此多的移动部件，即使是对代码的微小调整也可以引发显着的性能变化。这就是为什么本书中的第一个建议是：*始终衡量*。

\personal{我看到很多人在尝试优化他们的应用程序时都依赖直觉。通常，它会到处随机修复，而不会对应用程序的性能产生任何实际影响。}

没有经验的开发人员经常对源代码进行更改，并希望它能提高程序的性能。一个这样的例子是在整个代码库中用 `++i` 替换 `i++`，假设不使用之前的 `i` 值。在一般情况下，此更改不会对生成的代码产生任何影响，因为每个体面的优化编译器都会识别出之前的 i 值没有被使用，并且无论如何都会消除冗余副本。

世界各地流传的许多微优化技巧在过去是有效的，但现在的编译器已经学会了它们。此外，有些人倾向于过度使用传统的位旋转技巧。其中一个例子是使用 [基于 XOR 的交换习语](https://en.wikipedia.org/wiki/XOR_swap_algorithm)[^2]，而实际上，简单的 `std::swap` 会产生更快的代码。此类意外更改可能不会提高应用程序的性能。找到正确的修复位置应该是仔细性能分析的结果，而不是直觉和猜测。

有许多性能分析方法[^1] 可能会或可能不会引导您进行发现。本书中介绍的特定于 CPU 的性能分析方法有一个共同点：它们基于收集有关程序如何执行的某些信息。最终对程序源代码进行的任何更改都是通过分析和解释收集的数据来驱动的。

定位性能瓶颈只是工程师工作的一半。下半场是要妥善修复它。有时更改程序源代码中的一行可以显着提高性能。性能分析和调优都是关于如何找到和修复这条线的。错过这样的机会可能是一种巨大的浪费。

[^1]: Performance Analysis Methodology by B. Gregg - [http://www.brendangregg.com/methodology.html](http://www.brendangregg.com/methodology.html)
[^2]: XOR-based swap idiom - [https://en.wikipedia.org/wiki/XOR_swap_algorithm](https://en.wikipedia.org/wiki/XOR_swap_algorithm