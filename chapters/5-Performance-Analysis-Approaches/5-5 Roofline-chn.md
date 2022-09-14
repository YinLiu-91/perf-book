---
typora-root-url: ..\..\img
---

## Roofline Performance Model {#sec:roofline}

Roofline 性能模型于 2009 年在加州大学伯克利分校开发。它是一种面向吞吐量的性能模型，在 HPC 世界中被大量使用。该模型中的“roofline”表示应用程序的性能不能超过机器的能力这一事实。程序中的每个函数和每个循环都受到机器的计算或内存容量的限制。这个概念在图@fig:RooflineIntro 中表示：应用程序的性能将始终受到某个“屋顶线”功能的限制。

![车顶线模型。 *© 图片取自 [NERSC 文档](https://docs.nersc.gov/development/performance-debugging-tools/roofline/#arithmetic-intensity-ai-and-achieved-performance-flops-for-application-characterization ).*](/2/Roofline-intro.png){#fig:RooflineIntro width=80%}

硬件有两个主要限制：计算的速度（峰值计算性能，FLOPS）和移动数据的速度（峰值内存带宽，GB/s）。应用程序的最大性能受限于峰值 FLOPS（水平线）和平台带宽乘以算术强度（对角线）之间的最小值。图@fig:RooflineIntro 中显示的屋顶线图绘制了两个应用程序“A”和“B”相对于硬件限制的性能。程序的不同部分可能具有不同的性能特征。 Roofline 模型说明了这一点，并允许在同一图表上显示应用程序的多个功能和循环。

算术强度 (AI) 是 FLOPS 和字节之间的比率，可以为程序中的每个循环提取。让我们计算 [@lst:BasicMatMul] 中代码的算术强度。在最里面的循环体中，我们有一个加法和一个乘法；因此，我们有 2 个 FLOPS。另外，我们有三个读操作和一个写操作；因此，我们传输 `4 ops * 4 bytes = 16` 字节。该代码的算术强度为“2 / 16 = 0.125”。 AI 用作给定性能点的 X 轴上的值。

清单：朴素的并行矩阵乘法。

~~~~ {#lst:BasicMatMul .cpp .numberLines}
void matmul(int N, 浮动 a[][2048], 浮动 b[][2048], 浮动 c[][2048]) {
    #pragma omp 并行
    for(int i = 0; i < N; i++) {
        for(int j = 0; j < N; j++) {
            for(int k = 0; k < N; k++) {
                c[i][j] = c[i][j] + a[i][k] * b[k][j];
            }
        }
    }
}
～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～

提高应用程序性能的传统方法是充分利用机器的 SIMD 和多核功能。很多时候，我们需要优化很多方面：向量化、内存、线程。 Roofline 方法可以帮助评估您的应用程序的这些特征。在屋顶线图上，我们可以绘制标量单核、SIMD 单核和 SIMD 多核性能的理论最大值（参见图 @fig:RooflineIntro2）。这将使我们了解提高应用程序性能的空间。如果我们发现我们的应用程序是计算密集型的（即具有高算术强度）并且低于峰值标量单核性能，我们应该考虑强制矢量化（参见 [@sec:Vectorization]）并在多个线程之间分配工作.相反，如果应用程序的算术强度较低，我们应该寻找改进内存访问的方法（参见 [@sec:MemBound]）。使用 Roofline 模型优化性能的最终目标是提高分数。矢量化和线程将点向上移动，同时通过增加算术强度来优化内存访问，将点向右移动，也可能提高性能。

![屋顶线模型。](/2/Roofline-intro2.jpg){#fig:RooflineIntro2 width=70%}

理论最大值（屋顶线）可以根据您使用的机器的特性计算[^1]。通常，一旦您知道机器的参数，这并不难。对于 Intel Core i5-8259U 处理器，具有 AVX2 和 2 个 Fused Multiply Add (FMA) 单元的最大 FLOP（单精度浮点数）数可以计算为：
$$
\开始{对齐}
\textrm{Peak FLOPS} =& \textrm{ 8（逻辑核心数）}~\times~\frac{\textrm{256（AVX位宽）}}{\textrm{32位（浮点大小）}} 〜\次〜\
& \textrm{ 2 (FMA)} \times ~ \textrm{3.8 GHz (最大涡轮频率)} \\
& = \textrm{486.4 GFLOPs}
\end{对齐}
$$

我用于实验的 Intel NUC Kit NUC8i5BEH 的最大内存带宽可以计算为：
$$
\开始{对齐}
\textrm{峰值内存带宽} = &~\textrm{2400（DDR4 内存传输速率）}~\times~ \textrm{2（内存通道）} ~ \times \\ &~ \textrm{8（每次内存访问的字节数） )} ~ \times \textrm{1 (socket)}= \textrm{38.4 GiB/s}
\end{对齐}
$$

[Empirical Roofline Tool](https://bitbucket.org/berkeleylab/cs-roofline-toolkit/src/master/)[^2] 和 [Intel Advisor](https://software.intel.com/) 等自动化工具内容/www/us/en/develop/tools/advisor.html)[^3] (s

**Additional resources and links:** 

* NERSC Documentation, URL: [https://docs.nersc.gov/development/performance-debugging-tools/roofline/](https://docs.nersc.gov/development/performance-debugging-tools/roofline/).
* Lawrence Berkeley National Laboratory research, URL: [https://crd.lbl.gov/departments/computer-science/par/research/roofline/](https://crd.lbl.gov/departments/computer-science/par/research/roofline/)
* Collection of video presentations about Roofline model and Intel Advisor, URL: [https://techdecoded.intel.io/](https://techdecoded.intel.io/) (search "Roofline").
* `Perfplot` is a collection of scripts and tools that allow a user to instrument performance counters on a recent Intel platform, measure them, and use the results to generate roofline and performance plots. URL: [https://github.com/GeorgOfenbeck/perfplot](https://github.com/GeorgOfenbeck/perfplot)

[^1]: Note that often theoretical maximums are often presented in a device specification and can be easily looked up.
[^2]: Empirical Roofline Tool - [https://bitbucket.org/berkeleylab/cs-roofline-toolkit/src/master/](https://bitbucket.org/berkeleylab/cs-roofline-toolkit/src/master/).
[^3]: Intel Advisor - [https://software.intel.com/content/www/us/en/develop/tools/advisor.html](https://software.intel.com/content/www/us/en/develop/tools/advisor.html).
[^4]: Likwid - [https://github.com/RRZE-HPC/likwid](https://github.com/RRZE-HPC/likwid).
[^5]: Intel SDE - [https://software.intel.com/content/www/us/en/develop/articles/intel-software-development-emulator.html](https://software.intel.com/content/www/us/en/develop/articles/intel-software-development-emulator.html).
[^6]: See a more detailed comparison between methods of collecting roofline data in this presentation: [https://crd.lbl.gov/assets/Uploads/ECP20-Roofline-4-cpu.pdf](https://crd.lbl.gov/assets/Uploads/ECP20-Roofline-4-cpu.pdf).
