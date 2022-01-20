# Segmentation

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-segmentation.pdf)

- 如何支持一个更大的地址空间？

**分段**

> 在地址空间的每个逻辑段上设置一个基址寄存器和界地址寄存器，段仅仅是特定长度、连续的地址空间的一部分。

- 除了能够动态重定位，分段更好地支持离散地址空间
- 能实现代码段的共享

**问题**

- 分段会导致外部碎片
- 不够灵活



## 模拟实验

**Q1**：简单测试，是否它们生成的地址有效？

```bash
./segmentation.py -a 128 -p 512 -b 0 -l 20 -B 512 -L 20 -s 0
./segmentation.py -a 128 -p 512 -b 0 -l 20 -B 512 -L 20 -s 1
./segmentation.py -a 128 -p 512 -b 0 -l 20 -B 512 -L 20 -s 2
```

```bash
# 1 output
# VA 为 [0, 19] 的在 Segment 0 中
# VA 为 [108, 127] 的在 Segment 0 中
ARG seed 0
ARG address space size 128
ARG phys mem size 512

Segment register information:

  Segment 0 base  (grows positive) : 0x00000000 (decimal 0)
  Segment 0 limit                  : 20

  Segment 1 base  (grows negative) : 0x00000200 (decimal 512)
  Segment 1 limit                  : 20

Virtual Address Trace
  VA  0: 0x0000006c (decimal:  108) --> VALID in SEG1: 0x000001ec (decimal:  492)
  VA  1: 0x00000061 (decimal:   97) --> SEGMENTATION VIOLATION (SEG1)
  VA  2: 0x00000035 (decimal:   53) --> SEGMENTATION VIOLATION (SEG0)
  VA  3: 0x00000021 (decimal:   33) --> SEGMENTATION VIOLATION (SEG0)
  VA  4: 0x00000041 (decimal:   65) --> SEGMENTATION VIOLATION (SEG1)
```

另外两个类似做法



**Q2**：在 Q1 的基础上，段 0 的最大有效虚拟地址是多少？段 1 的最小有效虚拟地址是多少？整个地址空间最小的和最大的有效地址是多少？

> 段 0 的最大的有效虚拟地址为 19；段 0 的最大的有效虚拟地址为 108；0 和 127

```bash
./segmentation.py -a 128 -p 512 -b 0 -l 20 -B 512 -L 20 -A 0,19,109,127 -c
ARG seed 0
ARG address space size 128
ARG phys mem size 512

Segment register information:

  Segment 0 base  (grows positive) : 0x00000000 (decimal 0)
  Segment 0 limit                  : 20

  Segment 1 base  (grows negative) : 0x00000200 (decimal 512)
  Segment 1 limit                  : 20

Virtual Address Trace
  VA  0: 0x00000000 (decimal:    0) --> VALID in SEG0: 0x00000000 (decimal:    0)
  VA  1: 0x00000013 (decimal:   19) --> VALID in SEG0: 0x00000013 (decimal:   19)
  VA  2: 0x0000006d (decimal:  109) --> VALID in SEG1: 0x000001ed (decimal:  493)
  VA  3: 0x0000007f (decimal:  127) --> VALID in SEG1: 0x000001ff (decimal:  511)
```



**Q3**：在一个 128B 的物理内存中，有一个 16B 的地址空间，我们应该如何设置 `Base` 和 `Limit`，以便让模拟器为指定的虚拟地址生成以下以下结果：`valid, valid, violation, ..., violation, valid, valid`

> 虚拟地址为：`0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15`

```bash
./segmentation.py -a 16 -p 128 -A 0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15 --b0 0 --l0 2 --b1 16 --l1 2
```



**Q4**：假设我们想要生成这样一个问题，其中大约 90% 的随机生成的虚拟地址是有效的，那么如果设置模拟器的参数？哪些参数对于这个输出是重要的？

> 假设 `-a` 指定的参数为 `A`，那么随机产生的虚拟地址的范围为 `[0, A - 1]`
>
> 且 `Segment 0` 的有效地址范围为：`[0, limit_0 - 1]`；`Segment 1` 的有效地址范围为：`[A - limit_2 + 1, A -1]`
>
> 则 `1 - (limit_0 + limit_1 - 2) / A = 0.1`

```bash
./segmentation.py -a 1k -n 1000 --l0 450 --l1 450 -c
```



**Q5**：如何运行一个所有随机产生的虚拟地址都是无效的测试用例？

> 设置两个界地址寄存器的值为 `0` 即可

```bash
./segmentation.py -a 10 -p 100 --l0 0 --l1 0 -c
ARG seed 0
ARG address space size 10
ARG phys mem size 100

Segment register information:

  Segment 0 base  (grows positive) : 0x00000054 (decimal 84)
  Segment 0 limit                  : 0

  Segment 1 base  (grows negative) : 0x0000004b (decimal 75)
  Segment 1 limit                  : 0

Virtual Address Trace
  VA  0: 0x00000004 (decimal:    4) --> SEGMENTATION VIOLATION (SEG0)
  VA  1: 0x00000002 (decimal:    2) --> SEGMENTATION VIOLATION (SEG0)
  VA  2: 0x00000005 (decimal:    5) --> SEGMENTATION VIOLATION (SEG1)
  VA  3: 0x00000004 (decimal:    4) --> SEGMENTATION VIOLATION (SEG0)
  VA  4: 0x00000007 (decimal:    7) --> SEGMENTATION VIOLATION (SEG1)
```











