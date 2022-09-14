---
typora-root-url: ..\..\img
---

## UOPs (micro-ops) {#sec:sec_UOP}

具有 x86 架构的微处理器将复杂的 [CISC](https://en.wikipedia.org/wiki/Complex_instruction_set_computer)-like[^1] 指令转换为简单的 [RISC](https://en.wikipedia.org/wiki/ Reduced_instruction_set_computer)-like[^5] 微操作 - 缩写为 µops 或 uops。这样做的主要优点是 µops 可以乱序执行 [@fogMicroarchitecture，第 2.1 章]。一个简单的加法指令，例如 `ADD EAX,EBX` 只生成一个 µop，而更复杂的指令，例如 `ADD EAX,[MEM1]` 可能会生成两个：一个用于从内存读取到临时（未命名）寄存器，以及一种用于将临时寄存器的内容添加到“EAX”。指令“ADD [MEM1],EAX”可能产生三个微操作：一个用于从内存中读取，一个用于加法，一个用于将结果写回内存。指令之间的关系以及它们被拆分为微操作的方式可能因 CPU 代而异[^6]。

与将复杂的类似 CISC 的指令拆分成类似 RISC 的微操作（Uops）相反，后者也可以融合。现代英特尔 CPU 中有两种类型的融合：

* [Microfusion](https://easyperf.net/blog/2018/02/15/MicroFusion-in-Intel-CPUs)[^3] - 融合来自同一机器指令的微指令。微融合只能应用于两种类型的组合：内存写入操作和读取修改操作。例如：

  ```bash
  # Read the memory location [ESI] and add it to EAX
  # Two uops are fused into one at the decoding step.
  add    eax, [esi]
  ```
  
* [Macrofusion](https://easyperf.net/blog/2018/02/23/MacroFusion-in-Intel-CPUs)[^4] - 融合来自不同机器指令的微指令。 在某些情况下，解码器可以将算术或逻辑指令与随后的条件跳转指令融合到单个计算和分支微操作中。 例如：

  ```bash
  # Two uops from DEC and JNZ instructions are fused into one
  .loop:
    dec rdi
    jnz .loop
  ```

Micro- 和 Macrofusion 都在管道从解码到退役的所有阶段节省了带宽。 融合操作共享重排序缓冲区 (ROB) 中的单个条目。 当融合的微指令仅使用一个条目时，ROB 的容量会增加。 这个单一的 ROB 条目表示必须由两个不同的执行单元完成的两个操作。 融合的 ROB 条目被分派到两个不同的执行端口，但作为单个单元再次退役。 [@fogMicroarchitecture]

Linux `perf` 用户可以通过运行 [^2] 收集其工作负载的已发布、已执行和已停用的微指令数量：

```bash
$ perf stat -e uops_issued.any,uops_executed.thread,uops_retired.all -- a.exe
  2856278  uops_issued.any             
  2720241  uops_executed.thread
  2557884  uops_retired.all
```

可以在 [uops.info](https://uops.info/) 网站上找到有关最新 x86 微架构指令的延迟、吞吐量、端口使用和 uops 数量。

[^1]: CISC - [https://en.wikipedia.org/wiki/Complex_instruction_set_computer](https://en.wikipedia.org/wiki/Complex_instruction_set_computer).
[^2]: `UOPS_RETIRED.ALL` event is not available since Skylake. Use `UOPS_RETIRED.RETIRE_SLOTS`.
[^3]: UOP Microfusion - [https://easyperf.net/blog/2018/02/15/MicroFusion-in-Intel-CPUs](https://easyperf.net/blog/2018/02/15/MicroFusion-in-Intel-CPUs).
[^4]: UOP Macrofusion - [https://easyperf.net/blog/2018/02/23/MacroFusion-in-Intel-CPUs](https://easyperf.net/blog/2018/02/23/MacroFusion-in-Intel-CPUs).
[^5]: RISC - [https://en.wikipedia.org/wiki/Reduced_instruction_set_computer](https://en.wikipedia.org/wiki/Reduced_instruction_set_computer).
[^6]: However, for the latest Intel CPUs, the vast majority of instructions operating on registers generate exactly one uop.

