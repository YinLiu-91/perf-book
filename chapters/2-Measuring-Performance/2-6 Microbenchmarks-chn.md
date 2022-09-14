---
typora-root-url: ..\..\img
---

## Microbenchmarks

可以编写一个独立的微基准来快速测试一些假设。通常，微基准用于在优化某些特定功能时跟踪进度。几乎所有现代语言都有基准测试框架，C++ 使用 Google [benchmark](https://github.com/google/benchmark)[^3] 库，C# 有 [BenchmarkDotNet](https://github.com/dotnet /BenchmarkDotNet)[^4] 库，Julia 有 [BenchmarkTools](https://github.com/JuliaCI/BenchmarkTools.jl)[^5] 包，Java 有 [JMH](http://openjdk.java。 net/projects/code-tools/jmh/etc)[^6] (Java Microbenchmark Harness) 等

在编写微基准时，确保您要测试的场景在运行时由您的微基准实际执行是非常重要的。优化编译器可以消除可能使实验无用的重要代码，甚至更糟，使您得出错误的结论。在下面的示例中，现代编译器可能会消除整个循环：

```cp
// foo 不对字符串创建进行基准测试
无效 foo() {
  for (int i = 0; i < 1000; i++)
    std::string s("hi");
}
```

一个简单的测试方法是检查基准的配置文件，看看预期的代码是否作为热点突出。有时可以立即发现异常时序，因此在分析和比较基准运行时使用常识。防止编译器优化掉重要代码的流行方法之一是使用 [`DoNotOptimize`](https://github.com/google/benchmark/blob/c078337494086f9372a46b4ed31a3ae7b3f1a6a2/include/benchmark/benchmark.h#L307)- like[^7] 辅助函数，它在后台执行必要的内联汇编魔术：

```cp
// foo 对字符串创建进行基准测试
无效 foo() {
  for (int i = 0; i < 1000; i++) {
    std::string s("hi");
    不要优化；
  }
}
```

如果写得好，微基准可以成为性能数据的良好来源。它们通常用于比较关键功能的不同实现的性能。定义一个好的基准是它是否在将使用功能的实际条件下测试性能。如果基准测试使用的合成输入与实践中给出的不同，那么基准测试可能会误导您并导致您得出错误的结论。除此之外，当基准测试在没有其他要求苛刻的进程的系统上运行时，它拥有所有可用资源，包括 DRAM 和缓存空间。这样的基准测试可能会支持该函数的更快版本，即使它比其他版本消耗更多内存。但是，如果有相邻进程消耗大量 DRAM，则结果可能相反，这会导致属于基准进程的内存区域被交换到磁盘。

出于同样的原因，在得出对函数进行单元测试的结果时要小心。现代单元测试框架[^8] 提供了每个测试的持续时间。但是，此信息不能替代使用实际输入在实际条件下测试功能的精心编写的基准（请参阅 [@fogOptimizeCpp，第 16.2 章] 中的更多信息）。在实践中并不总是能够复制准确的输入和环境，但开发人员在编写良好的基准测试时应该考虑到这一点。

[^3]: Google benchmark library - [https://github.com/google/benchmark](https://github.com/google/benchmark)
[^4]: BenchmarkDotNet - [https://github.com/dotnet/BenchmarkDotNet](https://github.com/dotnet/BenchmarkDotNet)
[^5]: Julia BenchmarkTools - [https://github.com/JuliaCI/BenchmarkTools.jl](https://github.com/JuliaCI/BenchmarkTools.jl)
[^6]: Java Microbenchmark Harness - [http://openjdk.java.net/projects/code-tools/jmh/etc](http://openjdk.java.net/projects/code-tools/jmh/etc)
[^7]: For JMH, this is known as the `Blackhole.consume()`.
[^8]: For instance, GoogleTest ([https://github.com/google/googletest](https://github.com/google/googletest)).