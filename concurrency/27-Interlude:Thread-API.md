# Interlude: Thread API

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-api.pdf)

## CRUX

> 如何创建和控制线程？

```c++
// 使用线程 API 需要包含的头文件
#include <pthread.h>

// 创建一个线程的函数
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine)(void*), void *arg);

// 在 main 函数中调用下面这个函数会等待线程 thread 执行结束
int pthread_join(pthread_t thread, void **value_ptr);
```

**锁**

> 通过锁来保证线程互斥地访问临界区

```c++
// 对临界区上锁
int pthread_mutex_lock(pthread_mutex_t *mutex);

// 对临界区解锁
int pthread_mutex_unlock(pthread_mutex_t *mutex);

// 在锁使用之前必须被初始化
pthread_mutex_t lock;
int rc = pthread_mutex_init(&lock, NULL);
assert(rc == 0); // always check success!

// 另外两个版本的获取锁的函数
int pthread_mutex_trylock(pthread_mutex_t *mutex);
int pthread_mutex_timedlock(pthread_mutex_t *mutex, struct timespec *abs_timeout);
```

**条件变量**

> 用于相互合作的线程（一个线程只有等待另一个线程完成某件事之后才能继续执行），例如：生产者-消费者的场景中

```c++
// 让调用下面这个函数的线程休眠（必须先获得锁 mutex），等待其他线程唤醒它
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);

int pthread_cond_signal(pthread_cond_t *cond);

// 在条件变量使用之前必须被初始化
pthread_cond_t cond;
pthread_cond_init(&cond);
```

---

## 代码

首先编译这几个文件

```c++
cd ostep-homework/threads-api/
make
```



**Q1**：使用工具 `helgrind` 测试程序 `main-race`，看看它会打印什么信息？它是否会定位的正确的代码行？它给你了其他什么信息？

> 从下面可以看到，helgrind 报告了两个线程产生了 data race，查看源文件，发现确实是第 8 行和第 15 行

```bash
valgrind --tool=helgrind ./main-race

# output
==2186== Helgrind, a thread error detector    
==2186== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.
==2186== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==2186== Command: ./main-race     
==2186==
==2186== ---Thread-Announcement------------------------------------------        
==2186==     
==2186== Thread #1 is the program's root thread      
==2186==         
==2186== ---Thread-Announcement------------------------------------------
==2186==
==2186== Thread #2 was created
==2186==    at 0x518470E: clone (clone.S:71)
==2186==    by 0x4E4BEC4: create_thread (createthread.c:100)
==2186==    by 0x4E4BEC4: pthread_create@@GLIBC_2.2.5 (pthread_create.c:797)
==2186==    by 0x4C38A27: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==2186==    by 0x1087E2: main (main-race.c:14)
==2186==
==2186== ----------------------------------------------------------------
==2186==
==2186== Possible data race during read of size 4 at 0x309014 by thread #1
==2186== Locks held: none
==2186==    at 0x108806: main (main-race.c:15)
==2186==
==2186== This conflicts with a previous write of size 4 by thread #2
==2186== Locks held: none
==2186==    at 0x10879B: worker (main-race.c:8)
==2186==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==2186==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==2186==    by 0x518471E: clone (clone.S:95)
==2186==  Address 0x309014 is 0 bytes inside data symbol "balance"
==2186==
==2186== ----------------------------------------------------------------
==2186==
==2186== Possible data race during write of size 4 at 0x309014 by thread #1
==2186== Locks held: none
==2186==    at 0x10880F: main (main-race.c:15)
==2186==
==2186== This conflicts with a previous write of size 4 by thread #2
==2186== Locks held: none
==2186==    at 0x10879B: worker (main-race.c:8)
==2186==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==2186==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==2186==    by 0x518471E: clone (clone.S:95)
==2186==  Address 0x309014 is 0 bytes inside data symbol "balance"
==2186==
==2186==
==2186== For counts of detected and suppressed errors, rerun with: -v
==2186== Use --history-level=approx or =none to gain increased speed, at
==2186== the cost of reduced accuracy of conflicting-access information
==2186== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```



**Q2**：要是将 `main-race.c` 文件的第 8 行或者第 15 行去掉，再使用 `helgrind` 检查会怎么样？使用锁呢？

> 去掉该文件的第 8 行或者第 15 行后，`helgrind` 不会报错；只在一个地方使用锁 `helgrind` 还是会报错；需要在两个线程访问 `balance` 时加锁，然后再释放锁

```c++
#include <stdio.h>

#include "common_threads.h"

int balance = 0;
pthread_mutex_t lock;

void* worker(void* arg) {
    Mutex_lock(&lock);
    balance++; // unprotected access 
    Mutex_unlock(&lock);
    return NULL;
}

int main(int argc, char *argv[]) {
    Mutex_init(&lock);
    pthread_t p;
    Pthread_create(&p, NULL, worker, NULL);
    Mutex_lock(&lock);
    balance++; // unprotected access
    Mutex_unlock(&lock);
    Pthread_join(p, NULL);
    return 0;
}
```



**Q3**：现在运行程序 `main-deadlock`，会发生什么？

> 运行多次，有时候执行正常，有时候处于死锁状态



**Q4**：使用 `helgrind` 检查程序 `main-deadlock`，它会报告什么信息？

```bash
valgrind --tool=helgrind ./main-deadlock

# output
==7968== Helgrind, a thread error detector
==7968== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.
==7968== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==7968== Command: ./main-deadlock
==7968==
==7968== ---Thread-Announcement------------------------------------------
==7968==
==7968== Thread #3 was created
==7968==    at 0x518470E: clone (clone.S:71)
==7968==    by 0x4E4BEC4: create_thread (createthread.c:100)
==7968==    by 0x4E4BEC4: pthread_create@@GLIBC_2.2.5 (pthread_create.c:797)
==7968==    by 0x4C38A27: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x1089E8: main (main-deadlock.c:24)
==7968==
==7968== ----------------------------------------------------------------
==7968==
==7968== Thread #3: lock order "0x30A040 before 0x30A080" violated
==7968==
==7968== Observed (incorrect) order is: acquisition of lock at 0x30A080
==7968==    at 0x4C3603C: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x1088B6: worker (main-deadlock.c:13)
==7968==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==7968==    by 0x518471E: clone (clone.S:95)
==7968==
==7968==  followed by a later acquisition of lock at 0x30A040
==7968==    at 0x4C3603C: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x1088E5: worker (main-deadlock.c:14)
==7968==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==7968==    by 0x518471E: clone (clone.S:95)
==7968==
==7968== Required order was established by acquisition of lock at 0x30A040
==7968==    at 0x4C3603C: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x108858: worker (main-deadlock.c:10)
==7968==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==7968==    by 0x518471E: clone (clone.S:95)
==7968==
==7968==  followed by a later acquisition of lock at 0x30A080
==7968==    at 0x4C3603C: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x108887: worker (main-deadlock.c:11)
==7968==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==7968==    by 0x518471E: clone (clone.S:95)
==7968==
==7968==  Lock at 0x30A040 was first observed
==7968==    at 0x4C3603C: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x108858: worker (main-deadlock.c:10)
==7968==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==7968==    by 0x518471E: clone (clone.S:95)
==7968==  Address 0x30a040 is 0 bytes inside data symbol "m1"
==7968==
==7968==  Lock at 0x30A080 was first observed
==7968==    at 0x4C3603C: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x108887: worker (main-deadlock.c:11)
==7968==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==7968==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==7968==    by 0x518471E: clone (clone.S:95)
==7968==  Address 0x30a080 is 0 bytes inside data symbol "m2"
==7968==
==7968==
==7968==
==7968== For counts of detected and suppressed errors, rerun with: -v
==7968== Use --history-level=approx or =none to gain increased speed, at
==7968== the cost of reduced accuracy of conflicting-access information
==7968== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 7 from 7)
```



**Q5**：现在使用 `helgrind` 运行 `main-deadlock-global`，看看是否会出现和 Q4 一样的问题？

> `helgrind` 报告了和 Q4 中一样的错误

```bash
valgrind --tool=helgrind ./main-deadlock-global
```



**Q6**：程序 `main-signal` 使用一个变量 `done` 通知 child 已经完成，parent 现在可以继续执行。为什么这个程序效率低？

> 因为 parent 中有个循环会一直检测 `done` 的值，因此它会一直占用 CPU 时间



**Q7**：现在使用 `helgrind` 运行 `main-signal`，它会报告什么？

```bash
valgrind --tool=helgrind ./main-signal

# output
==12983== Helgrind, a thread error detector
==12983== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.
==12983== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==12983== Command: ./main-signal
==12983==
this should print first
==12983== ---Thread-Announcement------------------------------------------
==12983==
==12983== Thread #1 is the program's root thread
==12983==
==12983== ---Thread-Announcement------------------------------------------
==12983==
==12983== Thread #2 was created
==12983==    at 0x518470E: clone (clone.S:71)
==12983==    by 0x4E4BEC4: create_thread (createthread.c:100)
==12983==    by 0x4E4BEC4: pthread_create@@GLIBC_2.2.5 (pthread_create.c:797)
==12983==    by 0x4C38A27: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==12983==    by 0x1087DD: main (main-signal.c:15)
==12983==
==12983== ----------------------------------------------------------------
==12983==
==12983== Possible data race during read of size 4 at 0x309014 by thread #1
==12983== Locks held: none
==12983==    at 0x108802: main (main-signal.c:16)
==12983==
==12983== This conflicts with a previous write of size 4 by thread #2
==12983== Locks held: none
==12983==    at 0x108792: worker (main-signal.c:9)
==12983==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==12983==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==12983==    by 0x518471E: clone (clone.S:95)
==12983==  Address 0x309014 is 0 bytes inside data symbol "done"
==12983==
==12983== ----------------------------------------------------------------
==12983==
==12983== Possible data race during write of size 1 at 0x5C551A5 by thread #1
==12983== Locks held: none
==12983==    at 0x4C3E546: mempcpy (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==12983==    by 0x50EEA63: _IO_file_xsputn@@GLIBC_2.2.5 (fileops.c:1258)
==12983==    by 0x50E3B6E: puts (ioputs.c:40)
==12983==    by 0x108817: main (main-signal.c:18)
==12983==  Address 0x5c551a5 is 21 bytes inside a block of size 1,024 alloc'd
==12983==    at 0x4C32F2F: malloc (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==12983==    by 0x50E126B: _IO_file_doallocate (filedoalloc.c:101)
==12983==    by 0x50F1448: _IO_doallocbuf (genops.c:365)
==12983==    by 0x50F0567: _IO_file_overflow@@GLIBC_2.2.5 (fileops.c:759)
==12983==    by 0x50EEABC: _IO_file_xsputn@@GLIBC_2.2.5 (fileops.c:1266)
==12983==    by 0x50E3B6E: puts (ioputs.c:40)
==12983==    by 0x108791: worker (main-signal.c:8)
==12983==    by 0x4C38C26: ??? (in /usr/lib/valgrind/vgpreload_helgrind-amd64-linux.so)
==12983==    by 0x4E4B6DA: start_thread (pthread_create.c:463)
==12983==    by 0x518471E: clone (clone.S:95)
==12983==  Block was alloc'd by thread #2
==12983==
this should print last
==12983==
==12983== For counts of detected and suppressed errors, rerun with: -v
==12983== Use --history-level=approx or =none to gain increased speed, at
==12983== the cost of reduced accuracy of conflicting-access information
==12983== ERROR SUMMARY: 23 errors from 2 contexts (suppressed: 40 from 40)
```



**Q8**：程序 `main-signal-cv` 使用条件变量（以及锁）来唤醒 parent，这个程序的性能如何？

> 由于 parent 会被当 `dont` 不为 0 时会放弃 CPU，暂停执行，所以不会浪费 CPU 时间，改程序性能比较好



**Q9**：使用 `helgrind` 运行 `main-signal-cv`，它会报告什么？

```bash
valgrind --tool=helgrind ./main-signal-cv

# output
==15414== Helgrind, a thread error detector
==15414== Copyright (C) 2007-2017, and GNU GPL'd, by OpenWorks LLP et al.
==15414== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==15414== Command: ./main-signal-cv
==15414== 
this should print first
this should print last
==15414== 
==15414== For counts of detected and suppressed errors, rerun with: -v
==15414== Use --history-level=approx or =none to gain increased speed, at
==15414== the cost of reduced accuracy of conflicting-access information
==15414== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 7 from 7)
```