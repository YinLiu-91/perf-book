---
typora-root-url: ..\..\img
---

## Manual Performance Testing

如果工程师可以在开发过程中利用现有的性能测试基础设施，那就太好了。在上一节中，我们讨论了 CI 系统的一个很好的功能是可以向它提交性能评估作业。如果支持，则系统将返回测试开发人员想要提交到代码库的补丁的结果。由于各种原因，这可能并不总是可行，例如硬件不可用、设置对于测试基础设施而言过于复杂、需要收集额外的指标。在本节中，我们为本地绩效评估提供基本建议。

在我们的代码中进行性能改进时，我们需要一种方法来证明我们确实让它变得更好。此外，当我们提交常规代码更改时，我们希望确保性能不会倒退。通常，我们通过 1) 测量基线性能，2) 测量修改后程序的性能，以及 3) 将它们相互比较来做到这一点。在这种情况下，目标是比较同一功能程序的两个不同版本的性能。例如，我们有一个递归计算斐波那契数的程序，我们决定以迭代方式重写它。两者在功能上都是正确的并且产生相同的数字。现在我们需要比较两个程序的性能。

强烈建议不要只进行一次测量，而是多次运行基准测试。因此，我们有 N 个基线测量值和 N 个程序修改版本的测量值。现在我们需要一种方法来比较这两组测量值，以确定哪一组更快。这项任务本身就很棘手，并且有很多方法会被测量结果所迷惑，并可能从中得出错误的结论。如果你问任何数据科学家，他们会告诉你不应该依赖单一的指标（最小值/平均值/中值等）。

考虑为图@fig:CompDist 中的两个版本的程序收集的两种性能测量分布。该图表显示了我们获得给定版本程序的特定时间的概率。例如，版本“A”有约 32% 的机会在约 102 秒内完成。很容易说“A”比“B”快。然而，它只有在一定的概率“P”下才成立。这是因为“B”的某些测量值比“A”快。即使在“B”的所有测量值都比“A”的每个测量值慢的情况下，“P”的概率也不等于“100%”。这是因为我们总是可以为“B”生成一个额外的样本，这可能比“A”的某些样本要快。

![比较 2 个性能测量分布。](/1/CompDist2.png){#fig:CompDist width=80%}

使用分布图的一个有趣的优势是它可以让您发现基准测试中不需要的行为[^3]。如果分布是双峰的，则基准可能会经历两种不同类型的行为。双峰分布测量的一个常见原因是代码既有快速路径也有慢速路径，例如访问缓存（缓存命中与缓存未命中）和获取锁（竞争锁与非竞争锁）。为了“解决”这个问题，应该隔离不同的功能模式并分别进行基准测试。

数据科学家通常通过绘制分布来呈现测量结果，并避免计算加速比。这消除了有偏见的结论，并允许读者自己解释数据。绘制分布的流行方法之一是使用箱线图（见图@fig:BoxPlot），它允许在同一个图表上比较多个分布。

![箱形图。](/1/BoxPlot2.jpg){#fig:BoxPlot width=60%}

虽然可视化性能分布可以帮助您发现某些异常情况，但开发人员不应该使用它们来计算加速比。一般来说，很难通过查看性能测量分布来估计加速。此外，如上一节所述，它不适用于自动基准测试系统。通常，我们希望获得一个标量值，该值将表示程序的 2 个版本的性能分布之间的加速比，例如，“版本 `A` 比版本 `B` 快 `X%`”。

使用假设检验方法确定两个分布之间的统计关系。如果数据集之间的关系根据阈值概率（显着性等级）。如果分布[^11] 是高斯分布（[normal](https://en.wikipedia.org/wiki/Normal_distribution)[^5]），则使用参数假设检验（例如，[Student's T-test]（ https://en.wikipedia.org/wiki/Student's_t-test)[^7]) 来比较分布就足够了。如果要比较的分布不是高斯分布（例如，严重偏斜或多峰），则可以使用非参数测试（例如

[^1]: SPEC CPU 2017 benchmarks - [http://spec.org/cpu2017/Docs/overview.html#benchmarks](http://spec.org/cpu2017/Docs/overview.html#benchmarks)
[^2]: Standard deviation - [https://en.wikipedia.org/wiki/Standard_deviation](https://en.wikipedia.org/wiki/Standard_deviation)
[^3]: Another way to check this is to run the normality test: [https://en.wikipedia.org/wiki/Normality_test](https://en.wikipedia.org/wiki/Normality_test).
[^4]: This approach requires the number of measurements to be more than 1. Otherwise, the algorithm will stop after the first sample because a single run of a benchmark has `std.dev.` equals to zero.
[^5]: Normal distribution - [https://en.wikipedia.org/wiki/Normal_distribution](https://en.wikipedia.org/wiki/Normal_distribution).
[^6]: Null hypothesis - [https://en.wikipedia.org/wiki/Null_hypothesis](https://en.wikipedia.org/wiki/Null_hypothesis).
[^7]: Student's t-test - [https://en.wikipedia.org/wiki/Student%27s_t-test](https://en.wikipedia.org/wiki/Student's_t-test).
[^8]: Mann-Whitney U test - [https://en.wikipedia.org/wiki/Mann-Whitney_U_test](https://en.wikipedia.org/wiki/Mann-Whitney_U_test).
[^9]: Kruskal-Wallis analysis of variance - [https://en.wikipedia.org/wiki/Kruskal-Wallis_one-way_analysis_of_variance](https://en.wikipedia.org/wiki/Kruskal-Wallis_one-way_analysis_of_variance).
[^10]: Therefore, it is best used in Automated Testing Frameworks to verify that the commit didn't introduce any performance regressions.
[^11]: It is worth to mention that Gaussian distributions are very rarely seen in performance data. So, be cautious using formulas from statistics textbooks assuming Gaussian distributions. 
[^12]: Book "Workload Modeling for Computer Systems Performance Evaluation" - [https://www.cs.huji.ac.il/~feit/wlmod/](http://cs.huji.ac.il/~feit/wlmod/).

