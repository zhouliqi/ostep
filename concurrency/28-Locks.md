# Locks

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks.pdf)

**CRUX**

> - 如何构建一个高效的锁
> - 需要硬件什么支持？
> - 需要 OS 什么支持？

**评价指标**

- Lock 需要提供互斥操作
- 公平性：避免饥饿问题
- 性能

**Try**

- 使用中断

> 优点：简单
>
> 缺点：（1）需要调用的线程能够执行特权操作并信任该程序不会滥用中断（关闭中断该程序就会一直运行）；（2）该方法无法在多处理器的计算机上运行；（3）长时间关中断可能会导致中断丢失，这可能会导致严重的系统问题（4）这个方法效率很低



- 使用 Load-And-Store 测试 `flag` 标志

> 问题：（1）两个进程可能同时检测到 `flag` 标志位为 `0`，然后同时进入临界区，即该方法无法提供互斥操作。（2）自旋等待浪费了 CPU 时间（它不会主动放弃 CPU）



- 使用 Test-And-Set 实现自旋锁

> 由于前面两个方法存在的问题，因此构建一个 Lock 需要硬件的支持，即提供一个 test-and-set 的指令，它能实现**原子交换**；
>
> - 自旋锁实现了互斥操作
> - 但是自旋锁不能确保公平性，它可能会导致线程饥饿
> - 在单个 CPU 上，自旋锁的性能很差；在多个 CPU 上，它的性能很好（在临界区很短的时候，并且线程的个数和 CPU 的个数大致相等）

```c
// 这个操作是原子的
int TestAndSet(int *old_ptr, int new) {
	int old = *old_ptr; // fetch old value at old_ptr
	*old_ptr = new; 	// store ’new’ into old_ptr
	return old; 		// return the old value
}
```



- 使用 Compare-And-Swap（CAS）实现自旋锁

> 硬件提供的另一个 CAS 指令，它也是原子操作，它比 test-and-set 的指令更强大，例如在 **lock-free** 同步中

```c
int CompareAndSwap(int *ptr, int expected, int new) {
    int original = *ptr;
    if (original == expected) {
        *ptr = new;
    }
    return original;
}
```



- Load-Linked 和 Store-Conditional 指令

> 在 MIPS 架构中，可以使用这两个指令实现 Lock 和并发结构；Load-Linked 指令很像 load 指令，而 store-conditional 指令在没有其他线程取了这个地址的时候才会成功

```c
int LoadLinked(int *ptr) {
	return *ptr;
}

int StoreConditional(int *ptr, int value) {
    if (no update to *ptr since LoadLinked to this address) {
        *ptr = value;
        return 1; // success!
    } else {
    	return 0; // failed to update
    }
}
```



**CRUX**

> ​		由于**自旋锁**在大部分情况下的效率很低（例如 N 个线程在一个 CPU 上争用一把锁），因此：我们如何实现锁，它不会在 CPU 上自旋而浪费时间？通过上面的例子可以看出，只有硬件支持是无法实现的，此时还需要 OS 的支持

- Yield

> 在检测到锁已经被其他线程拥有的时候，此时主动放弃 CPU，但这种方法
>
> - 不是很高效（频繁的上下文切换）
> - 无法保证公平性

---

## 模拟实验

**Q1**：`flag.s` 汇编代码使用一个 `flag` 实现 Lock

> 理解其中的汇编代码



**Q2**：当使用默认参数运行 `flag.s` 时，它是否能正常运行？使用 `-M` 和 `-R` 标志跟踪变量和寄存器的变化，预测最后 `flag` 的值？

> 程序执行结束，`flag` 的值应该是 0，`count` 的值为 2；默认的中断时间是 50，足够 `flag.s` 执行完成。

```bash
./x86.py -p flag.s -R ax,bx -M flag,count -c
```



**Q3**：使用 `-a` 标志改变寄存器 `%bx` 的值，例如 `-a bx=2,bx=2`，结果又如何？

> 程序执行结束，`flag` 的值最终还是是 0，`count` 的值为 4

```bash
./x86.py -p flag.s -a bx=2,bx=2 -R ax,bx -M flag,count -c
```



**Q4**：为每个线程设置 `bx` 一个很高的值，并使用 `-i` 标志产生不同的中断时间，什么值将导致不正确的结果？什么值将产生正确的输出？

> 中断频率过低，越会产生不好的结果

```bash
./x86.py -p flag.s -a bx=10,bx=10 -R ax,bx -M flag,count -i 5 -c
```



**Q5**：现在，使用程序 `test-and-set.s`，首先看懂代码

> 它使用 `xchg` 指令实现一个简单的锁

```assembly
# 获取锁
.acquire
mov  $1, %ax        
xchg %ax, mutex     # atomic swap of 1 and mutex
test $0, %ax        # if we get 0 back: lock is free!
jne  .acquire       # if not, try again

# 释放锁
# release lock
mov  $0, mutex
```



**Q6**：现在运行 `test-and-set.s`，使用 `-i` 标志产生不同的中断时间，并且让循环运行多次；是否代码总能产生正确的结果？是否有时候会导致 CPU 的利用率低？

> 由于使用了锁，所以它总能产生正确的结果；有时会导致 CPU 的利用率不高

```bash
./x86.py -p test-and-set.s -a bx=10 -R ax,bx -M mutex,count -i 5 -c
```



**Q7**：使用 `-P` 来产生指定的锁代码的测试，

```bash
./x86.py -p test-and-set.s -i 10 -R ax,bx -M mutex,count -a bx=5 -P 1100111000111000111000 -c
```



**Q8**：阅读程序 `peterson.s`，它实现了 Peterson 算法



**Q9**：现在使用不同的 `-i` 值运行 `peterson.s`，你能看到什么不同？确保你正确地设置了线程的 ID

```bash
./x86.py -p peterson.s -R ax,bx,cx,fx -M flag,turn,count -a bx=0,bx=1 -i 5 -c
```



**Q10**：使用 `-P` 标志证明代码 work？

```bash
./x86.py -p peterson.s -R bx -M flag,turn,count -a bx=0,bx=1 -P 0000011111 -c
./x86.py -p peterson.s -R bx -M flag,turn,count -a bx=0,bx=1 -P 00000011111 -c
```



**Q11**：使用 `ticket.s` 来研究 ticket lock，是否它符合这一章节的代码？然后使用 `-a bx=1000,bx=1000` 运行该程序？看看发生了什么？是否线程花费了大量的时间在 spin-waiting？

> 线程没出现花很长时间自旋等待锁的情况

```bash
./x86.py -p ticket.s -M ticket,turn,count -a bx=1000,bx=1000 -i 5 -c
```



**Q12**：当你运行更多的线程时，该程序结果又如何？

```bash
./x86.py -p ticket.s -M ticket,turn,count -a bx=100,bx=100,bx=100 -i 5 -t 3 -c
```



**Q13**：接下来是 `yield.s` 程序，`yield` 指令会导致线程让出 CPU 的控制权，找到一个场景：`test-and-set.s` 程序会浪费 CPU 时间而在 spining，然而 `yield.s` 不会。它会节省多少指令？

```bash
./x86.py -p yield.s -M count,mutex -a bx=10,bx=10 -i 5 -c
```



**Q14**：最后是 `test-and-test-and-set.s` 程序，这个 lock 干了什么？与 `test-and-test.s` 相比，它节省了什么？

```assembly
# 获取锁的时候，多了
test $0, %ax
jne .acquire
mov  $1, %ax
```

> 因此大部分时间都是在第一个 `test` 处自旋，相比于 `xchg` 自旋对 CPU 的占用更少
