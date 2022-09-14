---
typora-root-url: ..\..\img
---

## Automated Detection of Performance Regressions

软件供应商试图增加部署频率正在成为一种趋势。公司不断寻求方法来加快将产品推向市场的速度。不幸的是，这并不意味着软件产品在每个新版本中都会变得更好。特别是，软件性能缺陷往往以惊人的速度泄漏到生产软件中 [@UnderstandingPerfRegress]。软件中的大量更改对分析所有这些结果和历史数据以检测性能回归提出了挑战。

软件性能回归是在软件从一个版本演变到下一个版本时错误地引入软件的缺陷。捕捉性能错误和改进意味着在存在来自测试基础设施的噪音的情况下检测哪些提交会改变软件的性能（通过性能测试来衡量）。从数据库系统到搜索引擎再到编译器，几乎所有大型软件系统在其持续演进和部署生命周期中都普遍经历了性能回归。在软件开发过程中完全避免性能回归可能是不可能的，但是通过适当的测试和诊断工具，可以将此类缺陷悄悄泄漏到生产代码中的可能性降到最低。

想到的第一个选项是：让人类查看图表并比较结果。我们想很快摆脱这种选择也就不足为奇了。人们往往会很快失去注意力，并且可能会错过回归，尤其是在嘈杂的图表上，如图 @fig:PerfRegress 所示。人类可能会捕捉到 8 月 5 日左右发生的性能回归，但人类是否会检测到后来的回归并不明显。除了容易出错之外，让人类参与循环也是一项耗时且无聊的工作，必须每天执行。

![4 次测试的性能趋势图，8 月 5 日性能略有下降（数值越高越好）。 *© 图片来自 [@MongoDBChangePointDetection]*](/1/PerfRegressions.png){#fig:PerfRegress width=90%}

第二种选择是有一个简单的阈值。它比第一种选择要好一些，但仍然有其自身的缺点。性能测试中的波动是不可避免的：有时，即使是无害的代码更改 [^3] 也会触发基准测试中的性能变化。为阈值选择正确的值非常困难，并且不能保证低误报率和误报率。将阈值设置得太低可能会导致分析一堆小回归，这些回归不是由源代码的变化引起的，而是由于一些随机噪声引起的。将阈值设置得太高可能会导致过滤掉真正的性能回归。小的变化可以慢慢堆积成更大的回归，这可能会被忽视[^1]。通过查看图@fig:PerfRegress，我们可以观察到阈值需要每次测试调整。可能适用于绿色（上行）测试的阈值不一定适用于紫色（下行）测试，因为它们具有不同级别的噪声。 [LUCI](https://chromium.googlesource.com/chromium/src.git/+/master/docs/tour_of_luci_ui.md)[LUCI](https://chromium.googlesource.com/chromium/src.git/+/master/docs/tour_of_luci_ui.md)[ ^2]，这是 Chromium 项目的一部分。

在 [@MongoDBChangePointDetection] 中采用了最近识别性能回归的方法之一。 MongoDB 开发人员实施了变更点分析，以识别其数据库产品不断发展的代码库中的性能变化。根据[@ChangePointAnalysis]，变化点分析是在时间有序的观察中检测分布变化的过程。 MongoDB 开发人员使用了一种“E-Divisive mean”算法，该算法通过分层选择将时间序列划分为集群的分布变化点来工作。他们的开源 CI 系统称为 [Evergreen](https://github.com/evergreen-ci/evergreen)[^4] 结合了该算法以在图表上显示更改点并打开 Jira 票证。关于这个自动化性能测试系统的更多细节可以在 [@Evergreen] 中找到。

[@AutoPerf] 中介绍了另一种有趣的方法。本文的作者介绍了“AutoPerf”，它使用硬件性能计数器（PMC，参见 [@sec:PMC]）来诊断修改程序中的性能回归。首先，它根据从原始程序收集的 PMC 配置文件数据来学习修改后函数的性能分布。然后，它根据从修改后的程序收集的 PMC 配置文件数据将性能偏差检测为异常。 `AutoPerf` 表明这种设计可以有效地诊断一些最复杂的软件性能错误，例如隐藏在并行程序中的错误。

不管底层算法如何

[^1]: E.g., suppose you have a threshold of 2%. If you have two consecutive 1.5% regressions, they both will be filtered out. But throughout two days, performance regression will sum up to 3%, which is bigger than the threshold.
[^2]: LUCI - [https://chromium.googlesource.com/chromium/src.git/+/master/docs/tour_of_luci_ui.md](https://chromium.googlesource.com/chromium/src.git/+/master/docs/tour_of_luci_ui.md)
[^3]: The following article shows that changing the order of the functions or removing dead functions can cause variations in performance: [https://easyperf.net/blog/2018/01/18/Code_alignment_issues](https://easyperf.net/blog/2018/01/18/Code_alignment_issues).
[^4]: Evergreen - [https://github.com/evergreen-ci/evergreen](https://github.com/evergreen-ci/evergreen).
