性能工程在高性能计算 (HPC)、云服务、高频交易 (HFT)、游戏开发和其他性能关键领域等行业不需要太多理由。例如，Google 报告说，搜索速度降低 2% 会导致每个用户的 [搜索次数减少 2%](https://assets.en.oreilly.com/1/event/29/Keynote Presentation 2.pdf)[^3]。对于雅虎！页面加载速度提高 400 毫秒导致 [流量增加 5-9%](https://www.slideshare.net/stoyan/dont-make-me-wait-or-building-highperformance-web-applications)[^4]。在大数字游戏中，小的改进可以产生重大影响。这样的例子证明，服务运行得越慢，使用它的人就越少。

有趣的是，性能工程不仅需要在上述领域。如今，在通用应用程序和服务领域也需要它。如果我们每天使用的许多工具未能满足其性能要求，它们将根本不存在。例如，集成到 Microsoft Visual Studio IDE 中的 Visual C++ [IntelliSense](https://docs.microsoft.com/en-us/visualstudio/ide/visual-cpp-intellisense)[^2] 功能具有非常严格的性能约束。要使 IntelliSense 自动完成功能起作用，它们必须以毫秒 [^5] 的顺序解析整个源代码库。如果需要几秒钟来建议自动完成选项，没有人会使用源代码编辑器。这样的功能必须非常灵敏，并在用户键入新代码时提供有效的延续。只有在设计软件时考虑到性能并考虑周到的性能工程，才能实现类似应用的成功。

有时，快速工具会在它们最初设计不适用的领域中找到用途。例如，如今，[Unreal](https://www.unrealengine.com)[^6] 和 [Unity](https://unity.com/)[^7] 等游戏引擎用于建筑、3d可视化、电影制作和其他领域。因为它们的性能如此之好，所以它们是需要 2d 和 3d 渲染、物理引擎、碰撞检测、声音、动画等应用程序的自然选择。

> “快速工具不仅能让用户更快地完成任务；它们允许用户以全新的方式完成全新类型的任务。” - Nelson Elhage 在他的博客 (2020) 上的 [文章](https://blog.nelhage.com/post/reflections-on-performance/)[^1] 中写道。

我希望不用说人们讨厌使用慢速软件。应用程序的性能特征可能是您的客户转向竞争对手产品的单一因素。通过强调性能，您可以为您的产品提供竞争优势。

性能工程是一项重要且有益的工作，但它可能非常耗时。事实上，性能优化是一场永无止境的游戏。总会有一些东西需要优化。不可避免地，开发商将达到收益递减点，进一步改进将以非常高的工程成本进行，并且可能不值得付出努力。从这个角度来看，知道何时停止优化是性能工作的一个关键方面[^8]。一些组织通过将这些信息集成到代码审查过程中来实现它：源代码行用相应的“成本”指标进行注释。使用这些数据，开发人员可以决定是否值得提高特定代码段的性能。

在开始性能调优之前，请确保您有充分的理由这样做。如果不能为您的产品增加价值，那么仅仅为了优化而进行的优化是没有用的。有意识的性能工程从明确定义的性能目标开始，说明您要实现的目标以及为什么要这样做。此外，如果您达到目标，您应该选择用于衡量的指标。您可以在 [@SystemsPerformance] 和 [@Akinshin2019] 中阅读有关设置性能目标的更多信息。

尽管如此，练习和掌握性能分析和调优的技能总是很棒的。如果您出于这个原因拿起了这本书，我们非常欢迎您继续阅读。