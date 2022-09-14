---
typora-root-url: ..\..\img
---

## Software and Hardware Timers {#sec:timers}

为了对执行时间进行基准测试，工程师通常使用所有现代平台都提供的两种不同的计时器：

 - **系统范围的高分辨率计时器**。这是一个系统计时器，通常实现为自某个任意开始日期以来发生的滴答数的简单计数，称为 [epoch](https://en.wikipedia.org/wiki/Epoch_(computing)) [^1]。这个时钟是单调的；即，它总是上升。系统计时器具有纳秒级分辨率[^9]，并且在所有 CPU 之间保持一致。它适用于测量持续时间超过一微秒的事件。系统时间可以通过系统调用[^2] 从操作系统中获取。系统范围的定时器与 CPU 频率无关。可以通过 `clock_gettime` 系统调用 [^10] 访问 Linux 系统上的系统计时器。在 C++ 中访问系统计时器的“事实上”标准是使用“std::chrono”，如 [@lst:Chrono] 所示。

   清单：使用 C++ std::chrono 访问系统计时器
   
   ~~~~ {#lst:Chrono .cpp}
   #include <cstdint>
   #include <时间>

   //以纳秒为单位返回经过的时间
   uint64_t timeWithChrono() {
     使用命名空间 std::chrono；
     uint64_t start = duration_cast<纳秒>
         (steady_clock::now().time_since_epoch()).count();
     //运行一些东西
     uint64_t end = duration_cast<纳秒>
         (steady_clock::now().time_since_epoch()).count();
     uint64_t delta = 结束 - 开始；
     返回增量；
   }
   ～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～
   
 - **时间戳计数器 (TSC)**。这是一个硬件定时器，它被实现为一个硬件寄存器。 TSC 是单调的并且具有恒定速率，即它不考虑频率变化。每个 CPU 都有自己的 TSC，它只是经过的参考周期数（参见 [@sec:secRefCycles]）。它适用于测量持续时间从纳秒到一分钟的短事件。 TSC 的值可以通过使用编译器内置函数 `__rdtsc` 来检索，如 [@lst:TSC] 所示，它在后台使用了 `RDTSC` 汇编指令。有关使用“RDTSC”汇编指令对代码进行基准测试的更多低级细节可以在白皮书 [@IntelRDTSC] 中访问。

   清单：使用 __rdtsc 编译器内置函数访问 TSC

   ~~~~ {#lst:TSC .cpp}
   #include <x86intrin.h>
   #include <cstdint>

   // 返回经过的参考时钟数
   uint64_t timeWithTSC() {
       uint64_t 开始 = __rdtsc();
       //运行一些东西
       返回 __rdtsc() - 开始；
   }
   ～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～～

选择使用哪个计时器非常简单，取决于您要测量的时间。如果您在非常短的时间内测量某物，TSC 将为您提供更好的准确性。相反，使用 TSC 来衡量运行数小时的程序是没有意义的。除非您真的需要循环精度，否则系统计时器应该足以应付大部分情况。请务必记住，访问系统计时器通常比访问 TSC 具有更高的延迟。进行“clock_gettime”系统调用可能比执行“RDTSC”指令慢十倍，后者需要 20 多个 CPU 周期。这对于最小化测量开销可能变得很重要，尤其是在生产环境中。 CppPerformanceBenchmarks 存储库的 [wiki 页面](https://gitlab.com/chriscox/CppPerformanceBenchmarks/-/wikis/ClockTimeAnalysis)[^3] 上提供了用于访问各种平台上的计时器的不同 API 的性能比较。

[^1]: Unix epoch starts at 1 January 1970 00:00:00 UT: [https://en.wikipedia.org/wiki/Unix_epoch](https://en.wikipedia.org/wiki/Unix_epoch).
[^2]: Retrieving system time - [https://en.wikipedia.org/wiki/System_time#Retrieving_system_time](https://en.wikipedia.org/wiki/System_time#Retrieving_system_time)
[^3]: CppPerformanceBenchmarks wiki - [https://gitlab.com/chriscox/CppPerformanceBenchmarks/-/wikis/ClockTimeAnalysis](https://gitlab.com/chriscox/CppPerformanceBenchmarks/-/wikis/ClockTimeAnalysis)
[^9]: Even though the system timer can return timestamps with nano-seconds accuracy, it is not suitable for measuring short running events because it takes a long time to obtain the timestamp via the `clock_gettime` system call.
[^10]: On Linux, one can query CPU time for each thread using the `pthread_getcpuclockid` system call.
