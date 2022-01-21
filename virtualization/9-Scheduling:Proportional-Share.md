# Scheduling: Proportional Share

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-lottery.pdf)

- Lottery Scheduling
- Stride Scheduling
- Completely Fair Scheduler (CFS) 

---

## 模拟实验

**Q1**：测试，两个任务，随机种子分别是：1,2,3

```bash
./lottery.py -j 3 -s 1 -c
./lottery.py -j 3 -s 2 -c
./lottery.py -j 3 -s 3 -c
```



**Q2**：运行两个任务，它们的运行时间都是 10，任务 0 的 ticket 为 1，而任务 1 的 ticket 为 100，当 ticket 的数量不平衡是会发生什么？任务 0 会在 任务 1 前完成吗？

> 任务 1 总是占有 CPU 的使用权，导致任务 0 一直得不到调度；可能；概率为 `1 / (10 ^ 12)`；

```bash
./lottery.py -l 10:1,10:100 -c
```



**Q3**：运行两个任务，它们的运行时间都是 10，它们的 ticket 相同且都是 100，调度程序有多不公平？运行多个随机数来测试它。不公平性取决于一个任务比另一个任务完成时间早多少。

```bash
# seed is 0, time slice is 1
./lottery.py -l 100:100,100:100 -s 0 -c | rg "DONE"
--> JOB 0 DONE at time 192
--> JOB 1 DONE at time 200

# seed is 11, time slice is 1
./lottery.py -l 100:100,100:100 -s 11 -c | rg "DONE"
--> JOB 0 DONE at time 196
--> JOB 1 DONE at time 200

# seed is 100, time slice is 1
./lottery.py -l 100:100,100:100 -s 11 -c | rg "DONE"
--> JOB 1 DONE at time 185
--> JOB 0 DONE at time 200
```



**Q4**：如果在 Q3 的基础上时间片的长度变长，那么结果又是什么？

> 不公平程度增加

```bash
# seed is 0, time slice is 10
./lottery.py -l 100:100,100:100 -q 10 -s 0 -c | rg "DONE"
--> JOB 1 DONE at time 150
--> JOB 0 DONE at time 200

# seed is 11, time slice is 10
./lottery.py -l 100:100,100:100 -s 11 -q 10 -c | rg "DONE"
--> JOB 0 DONE at time 160
--> JOB 1 DONE at time 200

# seed is 100, time slice is 10
./lottery.py -l 100:100,100:100 -s 100 -q 10 -c | rg "DONE"
--> JOB 1 DONE at time 140
--> JOB 0 DONE at time 200
```