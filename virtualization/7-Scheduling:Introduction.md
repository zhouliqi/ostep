# Scheduling: Introduction

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf)

> 评价指标：
>
> $T_{turnaround}=T_{completion}-T_{arrival}$
>
> $T_{response}=T_{firstrun}-T_{arrival}$

> 这一节主要介绍 OS 对进程的调度，OS 应该选择哪个进程运行？假设：
>
> - 每个任务运行相同的时间
> - 所有的任务在同一时刻到达
> - 一旦任务开始，那么它会运行直到它完成
> - 所有的任务只是用 CPU（没有 I/O 请求）
> - 每个任务的运行时间已知
>
> 我们会根据不同的假设讨论不同的调度策略

- 先来先服务（FCFS / FIFO）策略（非抢占式）
- 短作业优先（SJF）策略（非抢占式）
- 最短完成时间优先（STCF / PSJF）策略（抢占式）
- 时间片轮转（RR）策略



## 模拟实验

**Q1**：运行 3 个运行时间为 200 的任务，分别采用 SJF 和 FIFO 策略，计算它们的周转时间和响应时间？

> 因为 3 个任务的运行时间一样，所以 SJF 和 FIFO 输出相同

```bash
# SJF, FIFO
Execution trace:
  [ time   0 ] Run job 0 for 200.00 secs ( DONE at 200.00 )
  [ time 200 ] Run job 1 for 200.00 secs ( DONE at 400.00 )
  [ time 400 ] Run job 2 for 200.00 secs ( DONE at 600.00 )

Final statistics:
  Job   0 -- Response: 0.00  Turnaround 200.00  Wait 0.00
  Job   1 -- Response: 200.00  Turnaround 400.00  Wait 200.00
  Job   2 -- Response: 400.00  Turnaround 600.00  Wait 400.00

  Average -- Response: 200.00  Turnaround 400.00  Wait 200.00
```



**Q2**：条件和 Q1 一样，不过三个任务的运行时间变为 100,200,300，计算它们的周转时间和响应时间？

```bash
# SJF, FIFO
Execution trace:
  [ time   0 ] Run job 0 for 100.00 secs ( DONE at 100.00 )
  [ time 100 ] Run job 1 for 200.00 secs ( DONE at 300.00 )
  [ time 300 ] Run job 2 for 300.00 secs ( DONE at 600.00 )

Final statistics:
  Job   0 -- Response: 0.00  Turnaround 100.00  Wait 0.00
  Job   1 -- Response: 100.00  Turnaround 300.00  Wait 100.00
  Job   2 -- Response: 300.00  Turnaround 600.00  Wait 300.00

  Average -- Response: 133.33  Turnaround 333.33  Wait 133.33
```



**Q3**：条件和 Q1 一样，不过使用时间片为 1 的 RR 调度策略，计算它们的周转时间和响应时间？

```bash
# RR
Execution trace:
  [ time   0 ] Run job   0 for 1.00 secs
  [ time   1 ] Run job   1 for 1.00 secs
  [ time   2 ] Run job   2 for 1.00 secs
  ...
  [ time 597 ] Run job   0 for 1.00 secs ( DONE at 598.00 )
  [ time 598 ] Run job   1 for 1.00 secs ( DONE at 599.00 )
  [ time 599 ] Run job   2 for 1.00 secs ( DONE at 600.00 )

Final statistics:
  Job   0 -- Response: 0.00  Turnaround 598.00  Wait 398.00
  Job   1 -- Response: 1.00  Turnaround 599.00  Wait 399.00
  Job   2 -- Response: 2.00  Turnaround 600.00  Wait 400.00

  Average -- Response: 1.00  Turnaround 599.00  Wait 399.00
```



**Q4**：什么样的 workload 会使得 SJF 和 FIFO 调度策略的周转时间相同？

> 对于任务 $j_1,\ j_2,\cdots,\ j_n$，如果 $t_1\ \leq \ t_2\ \leq\cdots\leq \ t_n$，即运行时间短的任务先到，则二者相同



**Q5**：什么样的 workload 和时间片长度会使得 SJF 和 RR 调度策略的响应时间相同？

>对于任务 $j_1,\ j_2,\cdots,\ j_n$，如果 $t_1\ , \ t_2\ ,\cdots, \ t_n$，且 RR 的时间片长度为 $max\{t_1\ ,t_2,\ \cdots\ ,t_n \}$，则二者相同



**Q6**：随着任务的运行时间增加，对于 SJF 调度策略**响应时间**会怎么变？

> 变大

```bash
# 对于 4 个任务，其运行时间分别是：100, 200, 300, 400
Final statistics:
  Job   0 -- Response: 0.00  Turnaround 100.00  Wait 0.00
  Job   1 -- Response: 100.00  Turnaround 300.00  Wait 100.00
  Job   2 -- Response: 300.00  Turnaround 600.00  Wait 300.00
  Job   3 -- Response: 600.00  Turnaround 1000.00  Wait 600.00

  Average -- Response: 250.00  Turnaround 500.00  Wait 250.00

# 对于 4 个任务，其运行时间分别是：500, 600, 700, 800
Final statistics:
  Job   0 -- Response: 0.00  Turnaround 500.00  Wait 0.00
  Job   1 -- Response: 500.00  Turnaround 1100.00  Wait 500.00
  Job   2 -- Response: 1100.00  Turnaround 1800.00  Wait 1100.00
  Job   3 -- Response: 1800.00  Turnaround 2600.00  Wait 1800.00

  Average -- Response: 850.00  Turnaround 1500.00  Wait 850.00
```



**Q7**：随着时间片长度的增加，对于 RR 调度策略**响应时间**会怎么变？

> 变大。对于 N 个任务，如果时间片的长度为 $m$，且每个任务的运行时间都比 $m$ 大，那么响应时间的计算如下：
>
> $$\frac{\displaystyle\sum_{i=1}^{n}im}{N}=\frac{m\displaystyle\sum_{i=1}^{n}i}{N}=\frac{\frac{N(N+1)}{2}}{N}=\frac{m(N+1)}{2}$$