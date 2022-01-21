# Multiprocessor Scheduling

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched-multi.pdf)

前面，我们的调度策略都是基于单个 CPU 的，而如今的 CPU，通常由多个处理器（CPU），那么：

- OS 的调度程序如何在多个 CPU 上调度任务？
- 会出现什么问题？
- 我们前面介绍的调度策略是否还可以使用，或者需要提出新的调度策略？

> 单核和多核：总所周知，CPU 中存在缓存（例如：L1，L2，L3 级缓存），在单 CPU 中，不需要考虑数据不一致的问题，因为此时就一个核；然而在多处理器的计算机中，如何处理数据在 CPU 缓存中的不一致问题是很重要的。

---

## 模拟实验

**Q1**：在一个 CPU 上运行一个任务：运行时间为 30，`working-set size` 为 200，它多久才能完成？

```bash
./multi.py -n 1 -L a:30:200 -c -t

# output
Finished time 30
Per-CPU stats
  CPU 0  utilization 100.00 [ warm 0.00 ]
```



**Q2**：在 Q1 的基础上，现在 CPU 的 cache 大小为 300，那么它多久可以完成？

> 由于 `warm rate` 默认是 2x，那么这个任务在 20 的时候就可以完成

```bash
./multi.py -n 1 -L a:30:200 -M 300 -t -c
# output
Finished time 20
Per-CPU stats
  CPU 0  utilization 100.00 [ warm 50.00 ]
```



**Q3**：和 Q2 情况一样，但使用 `-T` 标志（`time_left tracing`）

```bash
./multi.py -n 1 -L a:30:200 -M 300 -T -t -c

Scheduler central queue: ['a']

   0   a [ 29]      
   1   a [ 28]      
   2   a [ 27]      
   3   a [ 26]      
   4   a [ 25]      
   5   a [ 24]      
   6   a [ 23]      
   7   a [ 22]      
   8   a [ 21]      
   9   a [ 20]      
----------------
  10   a [ 18]      
  11   a [ 16]      
  12   a [ 14]      
  13   a [ 12]      
  14   a [ 10]      
  15   a [  8]      
  16   a [  6]      
  17   a [  4]      
  18   a [  2]      
  19   a [  0]
```



**Q4**：情况和 Q3 一样，此时使用 `-C` 查看每个 CPU 对于每个任务的 cache 情况；当使用 `-w` 降低或者增加默认值时，会发生什么？

> `warmup_time` 的默认值为 10，当降低这个值时，运行时间将减少；反之将增加

```bash
./multi.py -n 1 -L a:30:200 -M 300 -T -t -C -c
# output
Scheduler central queue: ['a']

   0   a [ 29] cache[ ]     
   1   a [ 28] cache[ ]     
   2   a [ 27] cache[ ]     
   3   a [ 26] cache[ ]     
   4   a [ 25] cache[ ]     
   5   a [ 24] cache[ ]     
   6   a [ 23] cache[ ]     
   7   a [ 22] cache[ ]     
   8   a [ 21] cache[ ]     
   9   a [ 20] cache[w]     
-------------------------
  10   a [ 18] cache[w]     
  11   a [ 16] cache[w]     
  12   a [ 14] cache[w]     
  13   a [ 12] cache[w]     
  14   a [ 10] cache[w]     
  15   a [  8] cache[w]     
  16   a [  6] cache[w]     
  17   a [  4] cache[w]     
  18   a [  2] cache[w]     
  19   a [  0] cache[w]
```



**Q5**：现在有两个 CPU，任务为：`a:100:100,b:100:50,c:100:50`，使用 RR 策略，它们将运行多久？缓存的利用率如何？

> 3 个任务会在 2 的 CPU 中运行，则在 150 的时候 3 个任务都将结束；运行序列类似
>
> `CPU-0：a c b a c b a c b a c b a c b `
>
> `CPU-1：b a c b a c b a c b a c b a c `
>
> 所以缓存对于任务速度的提升影响比较小，因为某个任务不断地在 `CPU-0` 和 `CPU-1` 中切换（缓存将被清空）

```bash
./multi.py -n 2 -L a:100:100,b:100:50,c:100:50 -t -c

# output
Job name:a run_time:100 working_set_size:100
Job name:b run_time:100 working_set_size:50
Job name:c run_time:100 working_set_size:50

Scheduler central queue: ['a', 'b', 'c']
```



**Q6**：现在我们将使用 cache affinity（添加 `-A` 标志）

```bash
./multi.py -n 2 -L a:100:100,b:100:50,c:100:50 -A a:0,b:1,c:1 -T -t -C -c

# output
Finished time 110

Per-CPU stats
  CPU 0  utilization 50.00 [ warm 40.91 ]
  CPU 1  utilization 100.00 [ warm 81.82 ]
```



**Q7**：测试多处理器的缓存

```bash
# warmup_time 表示一个任务在某个 CPU 上执行这个时间后，这个任务会被缓存
# 每个 CPU 都有一个 cache size，表示可以容纳的缓存大小

# 缓存大小为 50，CPU 个数分别为为 1, 2, 3 个
./multi.py -n 1 -L a:100:100,b:100:100,c:100:100 -M 50 -T -t -C -c # Finished time 300
./multi.py -n 2 -L a:100:100,b:100:100,c:100:100 -M 50 -T -t -C -c # Finished time 150
./multi.py -n 3 -L a:100:100,b:100:100,c:100:100 -M 50 -T -t -C -c # Finished time 100

# 缓存大小为 100，CPU 个数分别为为 1, 2, 3 个
./multi.py -n 1 -L a:100:100,b:100:100,c:100:100 -M 100 -T -t -C -c # Finished time 300
./multi.py -n 2 -L a:100:100,b:100:100,c:100:100 -M 100 -T -t -C -c # Finished time 150
./multi.py -n 3 -L a:100:100,b:100:100,c:100:100 -M 100 -T -t -C -c # Finished time 55
```



**Q8**：测试每个 CPU 的调度选项（`-p` 标志），

```bash
./multi.py -n 2 -L a:100:100,b:100:50,c:100:50 -p 1 -T -t -C -c # Finished time 100
```



**Q9**：随机生成几个测试用例，预测它们的输出

```bash
./multi.py -n 2 -L a:50:100,b:100:100,c:100:100,d:100:100 -T -t -C -c # Finished time 180
./multi.py -n 3 -L a:50:100,b:100:100,c:100:100,d:100:100 -M 200 -T -t -C -c
```

