# Scheduling: The Multi-Level Feedback Queue

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-mlfq.pdf)

> ​		为了在没有先验知识的情况下**最小化周转时间和响应时间**，这一节介绍了**多级反馈队列**。MLFQ 由许多队列组成，每个队列都有一个不同的优先级。在任一时刻，将要运行的任务处于某个队列中，MLFQ 使用优先级来决定某一时刻运行那个任务：选择**优先级**最高的任务来运行。如果优先级最高的队列中有多个任务，那么对这个队列中的任务使用 RR 调度策略。
>
> - Rule #1：如果 `Priority(A) > Priority(B)`，那么任务 A 将被运行
> - Rule #2：如果 `Priority(A) == Priority(B)`，那么任务 A & B 使用 RR 策略运行
>
> ​        每个任务的优先级并不是一直不变的，例如：如果一个任务不断地放弃 CPU 而等待 IO，那么会提高它的优先级；而一直使用 CPU 的任务，则会降低它的优先级。
>
> - Rule #3：当系统刚启动一个任务时，会将它放到优先级最高的队列中
> - Rule #4a：如果一个任务在运行的时候用完了一整个时间片，那么它的优先级将被降低（例如：将这个任务移到下一个队列中）
> - Rule #4b：如果一个任务在时间片结束前放弃了 CPU 的使用，那么它将继续待到当前的队列中
>
> ​        为了避免 CPU 密集型的任务一直得不到运行的时间，需要添加规则
>
> - Rule #5：每经过一个周期 `S`，将系统中的所有任务移到优先级最高的队列中
>
> ​        为了阻止任务对调度程序的欺骗行为，例如：一个 CPU 密集型的程序故意在时间片用完前放弃 CPU，从而让自己继续在当前优先级的队列。我们修改了 Rule #4a 和 Rule #4b 成：
>
> - Rule #4：一旦一个任务在某个队列中用完了给它分配的时间片（不管它放弃 CPU 多少次），那么它的优先级将降低

---

## 模拟实验

**Q1**：运行几个随机测试，要求：2 个任务、2 个队列

```bash
# -m 限制每个任务的最大执行时间， -M 限制了每个任务的最大 IO 频率
./mlfq.py -q 3 -j 2 -n 2 -m 10 -M 0 -c

# output
Job List:
  Job  0: startTime   0 - runTime   8 - ioFreq   0
  Job  1: startTime   0 - runTime   4 - ioFreq   0

Execution Trace:
[ time 0 ] JOB BEGINS by JOB 0
[ time 0 ] JOB BEGINS by JOB 1
[ time 0 ] Run JOB 0 at PRIORITY 1 [ TICKS 9 ALLOT 1 TIME 7 (of 8) ]
[ time 1 ] Run JOB 0 at PRIORITY 1 [ TICKS 8 ALLOT 1 TIME 6 (of 8) ]
[ time 2 ] Run JOB 0 at PRIORITY 1 [ TICKS 7 ALLOT 1 TIME 5 (of 8) ]
[ time 3 ] Run JOB 0 at PRIORITY 1 [ TICKS 6 ALLOT 1 TIME 4 (of 8) ]
[ time 4 ] Run JOB 0 at PRIORITY 1 [ TICKS 5 ALLOT 1 TIME 3 (of 8) ]
[ time 5 ] Run JOB 0 at PRIORITY 1 [ TICKS 4 ALLOT 1 TIME 2 (of 8) ]
[ time 6 ] Run JOB 0 at PRIORITY 1 [ TICKS 3 ALLOT 1 TIME 1 (of 8) ]
[ time 7 ] Run JOB 0 at PRIORITY 1 [ TICKS 2 ALLOT 1 TIME 0 (of 8) ]
[ time 8 ] FINISHED JOB 0
[ time 8 ] Run JOB 1 at PRIORITY 1 [ TICKS 9 ALLOT 1 TIME 3 (of 4) ]
[ time 9 ] Run JOB 1 at PRIORITY 1 [ TICKS 8 ALLOT 1 TIME 2 (of 4) ]
[ time 10 ] Run JOB 1 at PRIORITY 1 [ TICKS 7 ALLOT 1 TIME 1 (of 4) ]
[ time 11 ] Run JOB 1 at PRIORITY 1 [ TICKS 6 ALLOT 1 TIME 0 (of 4) ]
[ time 12 ] FINISHED JOB 1

Final statistics:
  Job  0: startTime   0 - response   0 - turnaround   8
  Job  1: startTime   0 - response   8 - turnaround  12

  Avg  1: startTime n/a - response 4.00 - turnaround 10.00
```



**Q2**：复现这个节的每个 example？

```bash
# 8-2
./mlfq.py -n 3 --jlist 0,200,0 -q 10 -c

# 8-3
./mlfq.py -n 3 --jlist 0,200,0:100,20,0 -q 10 -c

# 8-4
./mlfq.py --jlist 0,180,0:100,20,1 -q 10 -c

# 8-5 left --- right
./mlfq.py --jlist 0,150,0:100,50,5:100,50,5 -n 3 -q 10 -i 5 -S -c
./mlfq.py --jlist 0,150,0:100,50,5:100,50,5 -n 3 -q 10 -i 5 -S -c -B 50

# 8.6 left --- right
./mlfq.py --jlist 0,200,0:30,200,9 -n 3 -q 10 -i 1 -S -c
./mlfq.py --jlist 0,200,0:30,200,9 -n 3 -q 10 -i 1 -c

# 8.7
./mlfq.py --jlist 0,100,0:0,100,0 -n 3 -Q 10,20,40 -c
```



**Q3**：如何设置 MLFQ 的参数，使得它和 RR 调度策略等价？

> 将队列设置为 1 即可

```bash
./mlfq.py --jlist 0,10,0:0,10,0 -n 1 -q 2 -c
```



**Q4**：设置一个工作场景：两个 jobs，MLFQ 的参数与规则 4a 和 4b 一样，并且在一个特定的时间间隔获得 99% 的 CPU 使用权

```bash
./mlfq.py --jlist 0,1000,0:300,1000,99 -n 3 -i 1 -S -c
```



**Q5**：优先级最高的队列的时间片长度为 10ms，为了保证一个 long-running 型的任务至少能得到 5% 的 CPU 使用时间，我们应该经过多久将任务调度回优先级最高的队列？

```bash
./mlfq.py --jlist 0,200,0:0,100,5:0,100,5 -n 3 -q 10 -i 5 -B 100 -c
```



**Q6**：使用 `-I` 标志查看调度行为

```bash
./mlfq.py --jlist 0,50,5:0,50,5:0,50,5 -n 3 -q 10 -i 5 -S -c
./mlfq.py --jlist 0,50,5:0,50,5:0,50,5 -n 3 -q 10 -i 5 -S -I -c
```

