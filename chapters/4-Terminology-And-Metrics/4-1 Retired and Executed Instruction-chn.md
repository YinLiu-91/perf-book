---
typora-root-url: ..\..\img
---

## Retired vs. Executed Instruction

现代处理器通常执行比程序流程所需的更多指令。发生这种情况是因为其中一些是推测性执行的，如 [@sec:SpeculativeExec] 中所述。对于通常的指令，一旦结果可用，CPU 就会提交结果，并且所有前面的指令都已经退役。但是对于推测性执行的指令，CPU 会保留它们的结果而不立即提交它们的结果。当推测结果正确时，CPU 会解除对此类指令的阻塞并正常进行。但是当推测发生错误时，CPU 会丢弃推测指令所做的所有更改，并且不会将其淘汰。因此，CPU处理的指令可以执行但不一定退出。考虑到这一点，我们通常可以预期执行指令的数量要高于退出指令的数量。[^1]

有一个固定性能计数器 (PMC)，用于收集退役指令的数量。它可以通过运行 Linux `perf` 轻松获得：

```bash
$ perf stat -e instructions ./a.exe
  2173414  instructions  #    0.80  insn per cycle 
# or just simply do:
$ perf stat ./a.exe
```

[^1]: Usually, a retired instruction has also gone through the execution stage, except those times when it does not require an execution unit. An example of it can be "MOV elimination" and "zero idiom". Read more on easyperf blog: [https://easyperf.net/blog/2018/04/22/What-optimizations-you-can-expect-from-CPU](https://easyperf.net/blog/2018/04/22/What-optimizations-you-can-expect-from-CPU). So, theoretically, there could be a case when the number of retired instructions is higher than the number of executed instructions.
