---
typora-root-url: ..\..\img
---

## Compiler Optimization Reports {#sec:compilerOptReports}

如今，软件开发非常依赖编译器来进行性能优化。编译器在加速我们的软件方面发挥着非常重要的作用。通常，开发人员将这项工作留给编译器，只有在他们看到有机会改进编译器无法完成的事情时才会干预。公平地说，这是一个很好的默认策略。为了更好的交互，编译器提供了开发人员可以用于性能分析的优化报告。

有时您想知道某个函数是否被内联或循环是否被矢量化、展开等。如果它被展开，展开因子是什么？有一个很难知道的方法：通过研究生成的汇编指令。不幸的是，并不是所有人都喜欢阅读汇编语言。如果函数很大，它调用其他函数或有许​​多也被矢量化的循环，或者如果编译器创建了同一个循环的多个版本，这可能会特别困难。幸运的是，大多数编译器，包括 `GCC`、`ICC` 和 `Clang`，都提供优化报告来检查对特定代码段进行了哪些优化。编译器提示的另一个示例是英特尔® [ISPC](https://ispc.github.io/ispc.html)[^3] 编译器（在 [@sec:ISPC] 中查看更多信息），它发出编译为相对低效代码的代码结构的性能警告数量。

[@lst:optReport] 显示了一个未由 `clang 6.0` 向量化的循环示例。

清单：交流

~~~~ {#lst:optReport .cpp .numberLines}
void foo(float* __restrict__ a,
         浮动* __restrict__ b,
         浮动* __restrict__ c,
         无符号 N) {
  for (无符号 i = 1; i < N; i++) {
    a[i] = c[i-1]; // 值是从上一次迭代中结转的
    c[i] = b[i];
  }
}
～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～

要在 clang 中发出优化报告，您需要使用 [-Rpass*](https://llvm.org/docs/Vectorizers.html#diagnostics) 标志：

```重击
$ clang -O3 -Rpass-analysis=.* -Rpass=.* -Rpass-missed=.* a.c -c
a.c:5:3：备注：循环未矢量化 [-Rpass-missed=loop-vectorize]
  for (无符号 i = 1; i < N; i++) {
  ^
a.c:5:3：备注：以 4 倍展开循环，运行时行程计数 [-Rpass=loop-unroll]
  for (无符号 i = 1; i < N; i++) {
  ^
```

通过检查上面的优化报告，我们可以看到循环没有向量化，而是展开了。开发人员在 [@lst:optReport] 的第 6 行的循环中识别向量依赖性的存在并不总是那么容易。 `c[i-1]` 加载的值取决于上一次迭代的存储（参见图@fig:VectorDep 中的操作#2 和#3）。可以通过手动展开循环的一些第一次迭代来揭示依赖关系：

```cp
// 迭代 1
  a[1] = c[0]；
  c[1] = b[1]; // 将值写入 c[1]
// 迭代 2
  a[2] = c[1]； // 读取 c[1] 的值
  c[2] = b[2];
...
```

![在 [@lst:optReport] 中可视化操作顺序。](/2/VectorDep.png){#fig:VectorDep width=50%}

如果我们要对 [@lst:optReport] 中的代码进行矢量化，则会导致在数组 `a` 中写入错误的值。假设一个 CPU SIMD 单元一次可以处理四个浮点数，我们将得到可以用以下伪代码表示的代码：

```cp
// 迭代 1
  a[1..4] = c[0..3]； // 哎呀，a[2..4] 得到了错误的值
  c[1..4] = b[1..4]；
...
```

[@lst:optReport] 中的代码无法向量化，因为循环内的操作顺序很重要。这个例子可以通过交换第 6 行和第 7 行来修复[^2]，而不改变函数的语义，如 [@lst:optReport2] 所示。有关使用编译器优化报告发现向量化机会和示例的更多信息，请参阅 [@sec:Vectorization]。

清单：交流

~~~~ {#lst:optReport2 .cpp .numberLines}
void foo(float* __restrict__ a,
         浮动* __restrict__ b,
         浮动* __restrict__ c,
         无符号 N) {
  for (无符号 i = 1; i < N; i++) {
    c[i] = b[i];
    a[i] = c[i-1];
  }
}
～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～

在优化报告中，我们可以看到循环现在已矢量化：

```重击
$ clang -O3 -Rpass-analysis=.* -Rpass=.* -Rpass-missed=.* a.c -c
a.cpp:5:3：备注：矢量化循环（矢量化宽度：4，交错计数：2）[-Rpass=loop-vectorize]
  for (无符号 i = 1; i < N; i++) {
  ^
```

每个源文件都会生成编译器报告，这可能非常大。用户可以简单地搜索输出中感兴趣的源行。 [编译器资源管理器](https://godbolt.org/)[^4] 网站具有用于基于 LLVM 的编译器的“优化输出”工具，当您将鼠标悬停在相应的源代码行上时，该工具会报告执行的转换。

在 LTO[^5] 模式下，在链接阶段进行了一些优化。要从编译和链接阶段发出编译器报告，应该将专用选项传递给编译器和链接器。有关更多信息，请参阅 LLVM“备注”[指南](https://llvm.org/docs/Remarks.html)[^6]

[^1]: Using compiler optimization pragmas - [https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts](https://easyperf.net/blog/2017/11/09/Multiversioning_by_trip_counts).
[^2]: Alternatively, the code can be improved by splitting the loop into two separate loops.
[^3]: ISPC - [https://ispc.github.io/ispc.html](https://ispc.github.io/ispc.html).
[^4]: Compiler Explorer - [https://godbolt.org/](https://godbolt.org/).
[^5]: Link-Time optimizations, also called InterProcedural Optimizations (IPO). Read more here: [https://en.wikipedia.org/wiki/Interprocedural_optimization](https://en.wikipedia.org/wiki/Interprocedural_optimization).
[^6]: LLVM compiler remarks - [https://llvm.org/docs/Remarks.html](https://llvm.org/docs/Remarks.html).
