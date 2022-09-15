---
typora-root-url: ..\..\img
---

## Last Branch Record {#sec:lbr}

现代 Intel 和 AMD CPU 具有称为 Last Branch Record (LBR) 的功能，其中 CPU 连续记录许多先前执行的分支。但在深入细节之前，有人可能会问：*为什么我们对分支如此感兴趣？* 嗯，因为这是我们能够确定程序控制流的方式。我们在很大程度上忽略了基本块中的其他指令（参见 [@sec:BasicBlock]），因为分支始终是基本块中的最后一条指令。由于基本块中的所有指令都保证执行一次，因此我们只能关注将“代表”整个基本块的分支。因此，如果我们跟踪每个分支的结果，就可以重建程序的整个逐行执行路径。事实上，这就是英特尔处理器跟踪 (PT) 功能能够做的事情，这将在 [@sec:secPT] 中讨论。 LBR 功能早于 PT，并且具有不同的用例和特殊功能。

由于 LBR 机制，CPU 可以在执行程序的同时将分支连续记录到一组特定于模型的寄存器 (MSR)，从而最大限度地降低速度[^15]。硬件记录每个分支的“from”和“to”地址以及一些额外的元数据（见图@fig:LbrAddr）。寄存器就像一个环形缓冲区，不断被覆盖，只提供 32 个最近的分支结果[^1]。如果我们收集足够长的源-目标对历史记录，我们将能够展开程序的控制流，就像深度有限的调用堆栈一样。

![LBR MSR 的 64 位地址布局。 *© 图片来自 [@IntelSDM].*](/4/LBR_ADDR.png){#fig:LbrAddr width=90%}

使用 LBR，我们可以对分支进行采样，但在每个采样期间，查看 LBR 堆栈中先前执行的分支。这可以合理地覆盖热代码路径中的控制流，但不会因为太多信息而使我们不知所措，因为只检查了较少数量的总分支。重要的是要记住，这仍然是采样，所以不是每个执行的分支都可以检查。 CPU 通常执行速度太快以至于不可行。[@LBR2016]

* **最后一个分支记录 (LBR) 堆栈** - 因为 Skylake 提供了 32 对 MSR，用于存储最近采用的分支的源地址和目标地址。
* **最后一个分支记录栈顶 (TOS) 指针** — 包含指向 LBR 堆栈中的 MSR 的指针，该堆栈包含最近记录的分支、中断或异常。

请务必记住，使用 LBR 机制仅记录采用的分支。下面的示例显示了如何在 LBR 堆栈中跟踪分支结果。

```asm
----> 4eda10:  mov    edi,DWORD PTR [rbx]
|     4eda12:  test   edi,edi
| --- 4eda14:  jns    4eda1e              <== NOT taken
| |   4eda16:  mov    eax,edi
| |   4eda18:  shl    eax,0x7
| |   4eda1b:  lea    edi,[rax+rdi*8]
| └─> 4eda1e:  call   4edb26
|     4eda23:  add    rbx,0x4
|     4eda27:  mov    DWORD PTR [rbx-0x4],eax
|     4eda2a:  cmp    rbx,rbp
 ---- 4eda2d:  jne    4eda10              <== taken
```

下面是我们在执行“CALL”指令时期望在 LBR 堆栈中看到的内容。 因为 `JNS` 分支 (`4eda14` -> `4eda1e`) 没有被采用，所以它没有被记录，因此不会出现在 LBR 堆栈中：

```
  FROM_IP        TO_IP
  ...            ...
  4eda2d         4eda10
  4eda1e         4edb26    <== LBR TOS
```

\personal{未记录的未使用分支可能会增加一些额外的分析负担，但通常不会使其过于复杂。 我们仍然可以展开 LBR 堆栈，因为我们知道控制流是从 TO\_IP(N-1) 到 FROM\_IP(N) 连续的。}

从 Haswell 开始，LBR 条目收到了额外的组件来检测分支预测错误。 在 LBR 条目中有一个专用位（参见 [@IntelSDM，第 3B 卷，第 17 章]）。 由于 Skylake 额外的“LBR_INFO”组件被添加到 LBR 条目中，该条目具有“周期计数”字段，用于计算自上次更新到 LBR 堆栈以来经过的核心时钟。 这些添加有重要的应用，我们将在后面讨论。 特定处理器的 LBR 条目的确切格式可以在 [@IntelSDM, Volume 3B, Chapters 17,18] 中找到。

用户可以通过执行以下命令确保在其系统上启用了 LBR：

```bash
$ dmesg | grep -i lbr
[    0.228149] Performance Events: PEBS fmt3+, 32-deep LBR, Skylake events, full-width counters, Intel PMU driver.
```

### Collecting LBR stacks

With Linux `perf`, one can collect LBR stacks using the command below:

```bash
$ ~/perf record -b -e cycles ./a.exe
[ perf record: Woken up 68 times to write data ]
[ perf record: Captured and wrote 17.205 MB perf.data (22089 samples) ]
```

LBR 堆栈也可以使用 `perf record --call-graph lbr` 命令收集，但收集的信息量少于使用 `perf record -b`。 例如，运行 `perf record --call-graph lbr` 时不会收集分支预测错误和循环数据。

因为每个收集的样本都捕获了整个 LBR 堆栈（最后 32 个分支记录），所以收集的数据（`perf.data`）的大小明显大于没有 LBR 的采样。 下面是可以用来转储收集到的分支堆栈内容的 Linux perf 命令：

```bash
$ perf script -F brstack &> dump.txt
```

If we look inside the `dump.txt` (it might be big) we will see something like as shown below:

```
...
0x4edabd/0x4edad0/P/-/-/2   0x4edaf9/0x4edab0/P/-/-/29
0x4edabd/0x4edad0/P/-/-/2   0x4edb24/0x4edab0/P/-/-/23
0x4edadd/0x4edb00/M/-/-/4   0x4edabd/0x4edad0/P/-/-/2
0x4edb24/0x4edab0/P/-/-/24  0x4edadd/0x4edb00/M/-/-/4
0x4edabd/0x4edad0/P/-/-/2   0x4edb24/0x4edab0/P/-/-/23
0x4edadd/0x4edb00/M/-/-/1   0x4edabd/0x4edad0/P/-/-/1
0x4edb24/0x4edab0/P/-/-/3   0x4edadd/0x4edb00/P/-/-/1
0x4edabd/0x4edad0/P/-/-/1   0x4edb24/0x4edab0/P/-/-/3
...
```

在上面的块中，我们展示了 LBR 堆栈中的 8 个条目，通常由 32 个 LBR 条目组成。 每个条目都有“FROM”和“TO”地址（十六进制值），预测标志（“M”/“P”）[^16]，以及周期数（每个条目最后位置的数字）。 标有“`-`”的组件与事务性内存（TSX）有关，这里我们不讨论。 好奇的读者可以在`perf script` [规范](http://man7.org/linux/man-pages/man1/perf-script.1.html)[^2]中查找解码的LBR条目的格式。

LBR 有许多重要的用例。 在接下来的部分中，我们将讨论最重要的部分。

### 捕获调用图

[@sec:secCollectCallStacks] 中讨论了收集调用图及其重要性。 LBR 可用于收集调用图信息，即使您编译的程序没有帧指针（由编译器选项 `-fomit-frame-pointer` 控制，默认为 ON）或调试信息 [^3]：

```bash
$ perf record --call-graph lbr -- ./a.exe
$ perf report -n --stdio
# Children   Self    Samples  Command  Object  Symbol       
# ........  .......  .......  .......  ......  .......
	99.96%  99.94%    65447    a.out    a.out  [.] bar
            |          
             --99.94%--main
                       |          
                       |--90.86%--foo
                       |          |          
                       |           --90.86%--bar
                       |          
                        --9.08%--zoo
                                  bar
```

如您所见，我们确定了程序中最热门的函数（即“bar”）。 此外，我们发现了在函数 `bar` 中花费最多时间的调用者（它是 `foo`）。 在这种情况下，我们可以看到 `bar` 中 91% 的样本都有 `foo` 作为其调用函数。[^4]

使用 LBR 特性，我们可以确定一个 Hyper Block（有时称为 Super Block），它是整个程序中执行频率最高的基本块链。 该链中的基本块不一定按顺序物理顺序放置； 它们按顺序执行。

### 识别热分支 {#sec:lbr_hot_branch}

LBR 功能还可以让我们知道最常使用的分支是什么：

```bash
$ perf record -e cycles -b -- ./a.exe
[ perf record: Woken up 3 times to write data ]
[ perf record: Captured and wrote 0.535 MB perf.data (670 samples) ]
$ perf report -n --sort overhead,srcline_from,srcline_to -F +dso,symbol_from,symbol_to --stdio
# Samples: 21K of event 'cycles'
# Event count (approx.): 21440
# Overhead  Samples  Object  Source Sym  Target Sym  From Line  To Line
# ........  .......  ......  ..........  ..........  .........  .......
  51.65%      11074   a.exe   [.] bar    [.] bar      a.c:4      a.c:5
  22.30%       4782   a.exe   [.] foo    [.] bar      a.c:10     (null)
  21.89%       4693   a.exe   [.] foo    [.] zoo      a.c:11     (null)
   4.03%        863   a.exe   [.] main   [.] foo      a.c:21     (null)
```

从这个例子中，我们可以看到超过 50% 的分支在 `bar` 函数内部，22% 的分支是从 `foo` 到 `bar` 的函数调用，等等。注意 `perf` 如何从 `cycles` 事件切换到分析 LBR 堆栈：只收集了 670 个样本，但每个样本都捕获了整个 LBR 堆栈。这为我们提供了 `670 * 32 = 21440` LBR 条目（分支结果）以供分析。[^5]

大多数情况下，仅从代码行和目标符号就可以确定分支的位置。但是，理论上，可以在一行上写两个“if”语句来编写代码。此外，在扩展宏定义时，所有扩展的代码都获得相同的源代码行，这是可能发生这种情况的另一种情况。这个问题并没有完全阻止分析，只是让它变得更加困难。为了消除两个分支的歧义，您可能需要自己分析原始 LBR 堆栈（参见 [easyperf](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability)[^6] 上的示例博客）。

### Analyze branch misprediction rate {#sec:secLBR_misp_rate}

It’s also possible to know the misprediction rate for hot branches [^7]:

```bash
$ perf record -e cycles -b -- ./a.exe
$ perf report -n --sort symbol_from,symbol_to -F +mispredict,srcline_from,srcline_to --stdio
# Samples: 657K of event 'cycles'
# Event count (approx.): 657888
# Overhead  Samples  Mis  From Line  To Line  Source Sym  Target Sym
# ........  .......  ...  .........  .......  ..........  ..........
    46.12%   303391   N   dec.c:36   dec.c:40  LzmaDec     LzmaDec   
    22.33%   146900   N   enc.c:25   enc.c:26  LzmaFind    LzmaFind  
     6.70%    44074   N   lz.c:13    lz.c:27   LzmaEnc     LzmaEnc   
     6.33%    41665   Y   dec.c:36   dec.c:40  LzmaDec     LzmaDec 
```

在这个例子中[^8]，对应于函数 `LzmaDec` 的行是我们特别感兴趣的。使用 [@sec:lbr_hot_branch] 的推理，我们可以得出结论，源代码行 dec.c:36 上的分支是基准测试中执行次数最多的分支。在 Linux `perf` 提供的输出中，我们可以发现两个对应于 `LzmaDec` 函数的条目：一个带有 `Y` 和一个带有 `N` 字母。一起分析这两个条目会给我们一个分支的误预测率。在这种情况下，我们知道“dec.c:36”行上的分支被预测了“303391”次（对应于“N”）并且被错误预测了“41665”次（对应于“Y”），这给了我们“ 88%`的预测率。

Linux `perf` 通过分析每个 LBR 条目并从中提取误预测位来计算误预测率。因此，对于每个分支，我们都有多次正确预测和错误预测。同样，由于采样的性质，某些分支可能有一个“N”条目但没有对应的“Y”条目。这可能意味着该分支没有 LBR 条目被错误预测，但这并不一定意味着预测率等于“100%”。

###机器码的精确计时{#sec:timed_lbr}

正如 [@sec:lbr] 中所讨论的，从 Skylake 架构开始，LBR 条目具有“循环计数”信息。这个额外的字段为我们提供了两个采取的分支之间经过的周期数。如果前一个 LBR 条目中的目标地址是某个基本块 (BB) 的开头，而当前 LBR 条目的源地址是同一基本块的最后一条指令，那么循环计数就是这个基本块的延迟。例如：

```
400618:   movb  $0x0, (%rbp,%rdx,1)    <= start of a BB
40061d:   add $0x1, %rdx 
400621:   cmp $0xc800000, %rdx 
400628:   jnz 0x400644                 <= end of a BB
```

Suppose we have two entries in the LBR stack:

```
  FROM_IP   TO_IP    Cycle Count
  ...       ...      ...
  40060a    400618    10
  400628    400644     5          <== LBR TOS
```

鉴于这些信息，我们知道在偏移量“400618”开始的基本块在 5 个周期内执行时发生过一次。如果我们收集到足够的样本，我们可以绘制该基本块延迟的概率密度函数（见图@fig:LBR_timing_BB）。该图表是通过分析所有满足上述规则的 LBR 条目来编制的。例如，基本块在大约 75 个周期内执行，只有 4% 的时间，但更常见的是，它在 260 到 314 个周期之间执行。这个块在一个不适合 CPU L3 缓存的巨大数组内有一个随机负载，所以基本块的延迟很大程度上取决于这个负载。图@fig:LBR_timing_BB 中显示了图表上有两个重要的峰值：第一个，大约 80 个周期对应于 L3 缓存命中，第二个峰值，大约 300 个周期，对应于加载请求全部发生的 L3 缓存未命中通往主内存的路。

![从地址 `0x400618` 开始的基本块延迟的概率密度函数。](/4/LBR_timing_BB.jpg){#fig:LBR_timing_BB width=90%}

此信息可用于进一步细粒度调整此基本块。这个例子可能受益于内存预取，我们将在 [@sec:memPrefetch] 中讨论。此外，此循环信息可用于定时循环迭代，其中每个循环迭代都以采用的分支（后沿）结束。

关于如何为任意基本块的延迟构建概率密度函数的示例可以在 [easyperf 博客](https://easyperf.net/blog/2019/04/03/Precise-timing-of-机器代码与 Linux 性能）[^9]。但是，在较新版本的 Linux perf 中，获取此信息要容易得多。例如[^7]：

```bash
$ perf record -e cycles -b -- ./a.exe
$ perf report -n --sort symbol_from,symbol_to -F +cycles,srcline_from,srcline_to --stdio
# Samples: 658K of event 'cycles'
# Event count (approx.): 658240
# Overhead  Samples  BBCycles  FromSrcLine  ToSrcLine   
# ........  .......  ........  ...........  ..........  
     2.82%   18581      1      dec.c:325    dec.c:326   
     2.54%   16728      2      dec.c:174    dec.c:174   
     2.40%   15815      4      dec.c:174    dec.c:174   
     2.28%   15032      2      find.c:375   find.c:376  
     1.59%   10484      1      dec.c:174    dec.c:174   
     1.44%   9474       1      enc.c:1310   enc.c:1315  
     1.43%   9392      10      7zCrc.c:15   7zCrc.c:17  
     0.85%   5567      32      dec.c:174    dec.c:174   
     0.78%   5126       1      enc.c:820    find.c:540  
     0.77%   5066       1      enc.c:1335   enc.c:1325  
     0.76%   5014       6      dec.c:299    dec.c:299   
     0.72%   4770       6      dec.c:174    dec.c:174   
     0.71%   4681       2      dec.c:396    dec.c:395   
     0.69%   4563       3      dec.c:174    dec.c:174   
     0.58%   3804      24      dec.c:174    dec.c:174   
```

从 `perf record` 的输出中删除了几行不重要的行，以使其适合页面。 如果我们现在关注源和目标是 `dec.c:174`[^10] 的分支，我们可以找到与其关联的多行。 Linux `perf` 首先按开销对条目进行排序，这需要我们手动过滤我们感兴趣的分支的条目。实际上，如果我们过滤它们，我们将获得以该分支结尾的基本块的延迟分布， 如表 {@tbl:bb_latency} 所示。 稍后我们可以绘制此数据并获得类似于图@fig:LBR_timing_BB 的图表。

----------------------------------------------
Cycles  Number of samples  Probability density
------  -----------------  -------------------
1       10484              17.0%

2       16728              27.1%

3       4563                7.4%

4       15815              25.6%

6       4770                7.7%

24      3804                6.2%

32      5567                9.0%
----------------------------------------------

Table: Probability density for basic block latency. {#tbl:bb_latency}

Currently, timed LBR is the most precise cycle-accurate source of timing information in the system.

### Estimating branch outcome probability

稍后在 [@sec:secFEOpt] 中，我们将讨论代码布局对性能的重要性。 再往前走一点，以跌落方式使用热路径[^11] 通常会提高程序的性能。 考虑到单个分支，知道 99% 的时间为假或真条件对于编译器做出更好的优化决策至关重要。

LBR 功能允许我们在不检测代码的情况下获取这些数据。 作为分析的结果，用户将获得条件的真假结果之间的比率，即分支被采用的次数和未采用的次数。 在分析间接跳转（switch 语句）和间接调用（虚拟调用）时，此功能尤为突出。 可以在 [easyperf 博客](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability)[^6] 上找到在实际应用程序中使用它的示例。

### Other use cases

* **配置文件引导优化**。 LBR 功能可以为优化编译器提供分析反馈数据。当考虑运行时开销时，与静态代码检测相比，LBR 可能是更好的选择。
* **捕获函数参数。** 当 LBR 功能与 PEBS 一起使用时（参见 [@sec:secPEBS]），可以捕获函数参数，因为根据 [x86 调用约定](https://en. wikipedia.org/wiki/X86_calling_conventions)[^12] 被调用者的前几个参数位于由 PEBS 记录捕获的寄存器中。 [@IntelSDM，附录 B，B.3.3.4 章]
* **基本块执行计数。** 由于分支 IP（源）和 LBR 堆栈中的前一个目标之间的所有基本块都只执行一次，因此可以评估程序内基本块的执行率。这个过程涉及构建每个基本块的起始地址的映射，然后向后遍历收集的 LBR 堆栈。 [@IntelSDM，附录 B，B.3.3.4 章]

[^1]: Only since Skylake microarchitecture. In Haswell and Broadwell architectures LBR stack is 16 entries deep. Check the Intel manual for information about other architectures.
[^2]: Linux `perf script` manual page - [http://man7.org/linux/man-pages/man1/perf-script.1.html](http://man7.org/linux/man-pages/man1/perf-script.1.html).
[^3]: Utilized by `perf record --call-graph dwarf`.
[^4]: We cannot necessarily drive conclusions about function call counts in this case. For example, we cannot say that `foo` calls `bar` 10 times more than `zoo`. It could be the case that `foo` calls `bar` once, but it executes some expensive path inside `bar` while `zoo` calls `bar` lots of times but returns quickly from it.
[^5]: The report header generated by perf confuses users because it says `21K of event cycles`. But there are `21K` LBR entries, not `cycles`.
[^6]: Analyzing raw LBR stacks - [https://easyperf.net/blog/2019/05/06/Estimating-branch-probability](https://easyperf.net/blog/2019/05/06/Estimating-branch-probability).
[^7]: Adding `-F +srcline_from,srcline_to` slows down building report. Hopefully, in newer versions of perf, decoding time will be improved.
[^8]: This example is taken from the real-world application, 7-zip benchmark: [https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip](https://github.com/llvm-mirror/test-suite/tree/master/MultiSource/Benchmarks/7zip). Although the output of perf report is trimmed a little bit to fit nicely on a page.
[^9]: Building a probability density function for the latency of an arbitrary basic block - [https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf](https://easyperf.net/blog/2019/04/03/Precise-timing-of-machine-code-with-Linux-perf).
[^10]: In the source code, line `dec.c:174` expands a macro that has a self-contained branch. That’s why the source and destination happen to be on the same line.
[^11]: I.e., when outcomes of branches are not taken.
[^12]: X86 calling conventions - [https://en.wikipedia.org/wiki/X86_calling_conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)
[^15]: Runtime overhead for the majority of LBR use cases is below 1%. [@Nowak2014TheOO]
[^16]: M - Mispredicted, P - Predicted.
