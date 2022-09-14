---
typora-root-url: ..\..\img
---

## Mispredicted branch {#sec:BbMisp}

现代 CPU 试图预测分支指令的结果（采用或不采用）。 例如，当处理器看到这样的代码时：

```bash
dec eax
jz .zero
# eax is not 0
...
zero:
# eax is 0
```

指令“jz”是一个分支指令，为了提高性能，现代 CPU 架构试图预测这样一个分支的结果。 这也称为“推测执行”。 处理器会推测，例如，不会采取分支，会执行对应于 `eax 不为 0` 的情况的代码。 但是，如果猜测错误，这称为“分支错误预测”，CPU 需要撤消它最近所做的所有推测工作。 这通常涉及 10 到 20 个时钟周期之间的惩罚。

Linux `perf` 用户可以通过运行以下命令检查分支错误预测的数量：

```bash
$ perf stat -e branches,branch-misses -- a.exe
   358209  branches
    14026  branch-misses #    3,92% of all branches        
# or simply do:
$ perf stat -- a.exe
```

\sectionbreak