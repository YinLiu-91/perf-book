---
typora-root-url: ..\..\img
---

## CPU Utilization

CPU 利用率是 CPU 在某个时间段内忙碌的时间百分比。 从技术上讲，当 CPU 没有运行内核“空闲”线程时，它被认为是使用的。

$$
CPU~Utilization = \frac{CPU\_CLK\_UNHALTED.REF\_TSC}{TSC},
$$

其中 `CPU_CLK_UNHALTED.REF_TSC` PMC 计算内核未处于停止状态时的参考周期数，`TSC` 代表时间戳计数器（在 [@sec:timers] 中讨论），它始终在滴答作响。

如果 CPU 利用率低，通常意味着应用程序的性能不佳，因为 CPU 浪费了一部分时间。 然而，高 CPU 利用率也并不总是好的。 这表明系统正在做一些工作，但并没有准确地说明它在做什么：即使 CPU 在等待内存访问时处于停滞状态，它也可能被高度利用。 在多线程上下文中，线程也可以在等待资源继续进行时旋转，因此有效的 CPU 利用率可以过滤旋转时间（参见 [@sec:secMT_metrics]）。

Linux `perf` 自动计算系统上所有 CPU 的 CPU 利用率：

```bash
$ perf stat -- a.exe
  0.634874  task-clock (msec) #    0.773 CPUs utilized   
```
