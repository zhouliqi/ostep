# Paging: Introduction

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-paging.pdf)

**分页**

> 将物理内存空间划分成一个个**固定大小**的单元（页），我们把这种思想叫做分页

**CRUX**

- 如何用页面虚拟化内存，以避免分段的问题？
- 基本方法是什么？
- 如果最小化时间和空间开销，同时得这些方法表现良好？

**页表**

- 如何减少页表占用的内存空间？（多级页表）
- 如何减少性能的降低？



---

## 模拟实验

**Q1**：`-P 1k` 时，如果虚拟地址空间增加，则页表的大小怎么变？`-a 1m` 时，随着页面大小增大，则页表的大小怎么变？

> 变大；变小；随着地址空间的增加，页表的大小会增大；随着页面大小的增加，页表的大小会减少；使用大的页表浪费空间

```bash
./paging-linear-translate.py -P 1k -a 1m -p 512m -v -n 0 # page table entry: 1024
./paging-linear-translate.py -P 1k -a 2m -p 512m -v -n 0 # page table entry: 2048
./paging-linear-translate.py -P 1k -a 4m -p 512m -v -n 0 # page table entry: 4096

./paging-linear-translate.py -P 1k -a 1m -p 512m -v -n 0 # page table entry: 1024
./paging-linear-translate.py -P 2k -a 1m -p 512m -v -n 0 # page table entry: 512
./paging-linear-translate.py -P 4k -a 1m -p 512m -v -n 0 # page table entry: 256
```



**Q2**：增加分配的页面的比例会发生什么？

> VA 的有效率增加

```bash
./paging-linear-translate.py -P 1k -a 16k -p 32k -v -u 0
./paging-linear-translate.py -P 1k -a 16k -p 32k -v -u 25
./paging-linear-translate.py -P 1k -a 16k -p 32k -v -u 50
./paging-linear-translate.py -P 1k -a 16k -p 32k -v -u 75
./paging-linear-translate.py -P 1k -a 16k -p 32k -v -u 100
```



**Q3**：下面哪些参数的组合是不现实的，为什么？

> 都不太合理：1 和 2 页面数太少；3 单个页面太大

```bash
./paging-linear-translate.py -P 8 -a 32 -p 1024 -v -s 1
./paging-linear-translate.py -P 8k -a 32k -p 1m -v -s 2
./paging-linear-translate.py -P 1m -a 256m -p 512m -v -s 3
```



**Q4**：如果虚拟空间大小物理空间大小会发生什么？

> 报错

```bash
./paging-linear-translate.py -P 1k -a 16k -p 4k -v

# output
ARG seed 0
ARG address space size 16k
ARG phys mem size 4k
ARG page size 1k
ARG verbose True
ARG addresses -1

Error: physical memory size must be GREATER than address space size (for this simulation)
```