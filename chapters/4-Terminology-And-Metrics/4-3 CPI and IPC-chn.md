---
typora-root-url: ..\..\img
---

## CPI & IPC {#sec:IPC}

这是两个非常重要的指标：

* 每条指令的周期数 (CPI) - 平均退出一条指令需要多少个周期。
* 每周期指令数 (IPC) - 平均每个周期有多少指令被淘汰。

$$
IPC = \frac{INST\_RETIRED.ANY}{CPU\_CLK\_UNHALTED.THREAD}
$$

$$
CPI = \frac{1}{IPC},
$$

其中 `INST_RETIRED.ANY` PMC 计算退出指令的数量，`CPU_CLK_UNHALTED.THREAD` 计算线程未处于停止状态时的核心周期数。

基于这些指标可以进行多种类型的分析。 它对于评估硬件和软件效率都很有用。 硬件工程师使用此指标来比较 CPU 世代和来自不同供应商的 CPU。 SW 工程师在优化应用程序时会考虑 IPC 和 CPI。 总的来说，我们希望有低 CPI 和高 IPC。 Linux `perf` 用户可以通过运行以下命令了解 IPC 的工作负载：

```bash
$ perf stat -e cycles,instructions -- a.exe
  2369632  cycles                               
  1725916  instructions  #    0,73  insn per cycle
# or just simply do:
$ perf stat ./a.exe
```