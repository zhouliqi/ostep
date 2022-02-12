

# Concurrency: An Introduction

> [讲义链接](http://www.cs.wisc.edu/~remzi/OSTEP/threads-intro.pdf)

- 为什么要使用多线程？

> ​		为了提高程序的执行效率（并行）

- 多线程情况下会遇到什么问题？

> ​		由于同一个进程内的线程共享地址空间，因此当多个线程同时访问一个共享数据时，如果不采取其他的机制，则会产生 race condition 导致程序运行的结果和预期不符的情况

**CRUX**

> 为了构建一个能够使用的同步原语
>
> - 硬件需要完成什么？
> - OS 又需要完成什么？
> - 如何正确且高效地构建这些原语？
> - 程序如何使用这些原语来得到想要的结果？

---

## 模拟实验

**Q1**：运行 `./x86.py -p loop.s -t 1 -i 100 -R dx`，则寄存器 `%dx` 的值如何变化？

```bash
./x86.py -p loop.s -t 1 -i 100 -R dx

# output
   dx          Thread 0         
    0   
   -1   1000 sub  $1,%dx
   -1   1001 test $0,%dx
   -1   1002 jgte .top
   -1   1003 halt
```



**Q2**：运行命令 `./x86.py -p loop.s -t 2 -i 100 -a dx=3,dx=3 -R dx`，它会指定两个线程，初始化每个线程的 `%dx` 为 3，则寄存器 `%dx` 的值如何变化？是否有数据争用？

> 线程有其私有的寄存器值；由于中断的时间比较长，所以没有数据争用

```bash
./x86.py -p loop.s -t 2 -i 100 -a dx=3,dx=3 -R dx

# output
   dx          Thread 0                Thread 1
    3
    2   1000 sub  $1,%dx
    2   1001 test $0,%dx
    2   1002 jgte .top
    1   1000 sub  $1,%dx
    1   1001 test $0,%dx
    1   1002 jgte .top
    0   1000 sub  $1,%dx
    0   1001 test $0,%dx
    0   1002 jgte .top
   -1   1000 sub  $1,%dx
   -1   1001 test $0,%dx
   -1   1002 jgte .top
   -1   1003 halt
    3   ----- Halt;Switch -----  ----- Halt;Switch -----
    2                            1000 sub  $1,%dx
    2                            1001 test $0,%dx
    2                            1002 jgte .top
    1                            1000 sub  $1,%dx
    1                            1001 test $0,%dx
    1                            1002 jgte .top
    0                            1000 sub  $1,%dx
    0                            1001 test $0,%dx
    0                            1002 jgte .top
   -1                            1000 sub  $1,%dx
   -1                            1001 test $0,%dx
   -1                            1002 jgte .top
   -1                            1003 halt

```



**Q3**：运行命令 `./x86.py -p loop.s -t 2 -i 3 -r -a dx=3,dx=3 -R dx`，它会使中断间隔变短并且随机，使用不用的 `-s` 参数来测试不同的执行序列

```bash
./x86.py -p loop.s -t 2 -i 3 -r -a dx=3,dx=3 -R dx -s 0 -c
./x86.py -p loop.s -t 2 -i 3 -r -a dx=3,dx=3 -R dx -s 1 -c
./x86.py -p loop.s -t 2 -i 3 -r -a dx=3,dx=3 -R dx -s 2 -c
```



**Q4**：使用程序 `looping-race-nolock.s`，它会访问位于地址 2000 的共享变量。运行命令 `./x86.py -p looping-race-nolock.s -t 1 -M 2000` 查看该值的在运行时的情况

```bash
./x86.py -p looping-race-nolock.s -t 1 -M 2000

# output
 2000          Thread 0         
    0   
    0   1000 mov 2000, %ax
    0   1001 add $1, %ax
    1   1002 mov %ax, 2000
    1   1003 sub  $1, %bx
    1   1004 test $0, %bx
    1   1005 jgt .top
    1   1006 halt
```



**Q5**：在 Q4 的基础上运行多个线程，即 `./x86.py -p looping-race-nolock.s -t 2 -a bx=3 -M 2000`，为什么每个线程循环了三次？最终的 value 是多少？

> 因为 `%bx` 设为了 3，循环退出的条件是 `%bx` 小于 0，所以每个线程循环了三次；最终 value 为：6

```bash
./x86.py -p looping-race-nolock.s -t 2 -a bx=3 -M 2000
```



**Q6**：运行命令 `./x86.py -p looping-race-nolock.s -t 2 -M 2000 -i 4 -r -s 0`，使用不用的 `-s` 参数。则最终 value 的值分别是多少？

> 只要 `1000,1001,1002` 这三条指令在时钟中断前完成，则不会影响最终结果

```bash
./x86.py -p looping-race-nolock.s -t 2 -M 2000 -i 4 -r -s 0 -c # 2
./x86.py -p looping-race-nolock.s -t 2 -M 2000 -i 4 -r -s 1 -c # 1
./x86.py -p looping-race-nolock.s -t 2 -M 2000 -i 4 -r -s 2 -c # 2
```



**Q7**：运行命令 `./x86.py -p looping-race-nolock.s -a bx=1 -t 2 -M 2000 -i 1`，则最终 value 的值是多少？当改变 `-i` 为 2 或者 3 时呢？哪些时钟中断间隔能保证程序执行后得到正确的结果？

> `-i` 大于 3 即可

```bash
./x86.py -p looping-race-nolock.s -a bx=1 -t 2 -M 2000 -i 1 # 1
./x86.py -p looping-race-nolock.s -a bx=1 -t 2 -M 2000 -i 2 # 1
./x86.py -p looping-race-nolock.s -a bx=1 -t 2 -M 2000 -i 3 # 2
```



**Q8**：让循环运行更多次，例如设置 `-a bx=100`，那么 `-i` 设置多少才能产生正确的结果？

```bash
./x86.py -p looping-race-nolock.s -a bx=100 -t 2 -M 2000 -i 1
```



**Q9**：运行 `./x86.py -p wait-for-me.s -a ax=1,ax=0 -R ax -M 2000`，它会给线程 0 设置 `%ax` 的值为 1，给线程 1 的 `%ax` 的值设为 0，并且观测 `%ax` 和 地址 `2000` 值的变化，则结果是什么？终止 value 的值是多少？

```bash
./x86.py -p wait-for-me.s -a ax=1,ax=0 -R ax -M 2000

# output
 2000      ax          Thread 0                Thread 1         
    0       1   
    0       1   1000 test $1, %ax
    0       1   1001 je .signaller
    1       1   1006 mov  $1, 2000
    1       1   1007 halt
    1       0   ----- Halt;Switch -----  ----- Halt;Switch -----  
    1       0                            1000 test $1, %ax
    1       0                            1001 je .signaller
    1       0                            1002 mov  2000, %cx
    1       0                            1003 test $1, %cx
    1       0                            1004 jne .waiter
    1       0                            1005 halt
```



**Q10**：运行 `./x86.py -p wait-for-me.s -a ax=0,ax=1 -R ax -M 2000`，则如果变化？

> 线程 0 一直在循环检测 2000 处的值直到中断的发生；改变时间间隔，则循环的次数也会增加

```bash
./x86.py -p wait-for-me.s -a ax=0,ax=1 -R ax -M 2000
./x86.py -p wait-for-me.s -a ax=0,ax=1 -R ax -M 2000 -i 1000
```
