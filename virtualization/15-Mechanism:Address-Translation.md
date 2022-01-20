# Mechanism: Address Translation

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-mechanism.pdf)

**CRUX**

- 如何构建一个高性能的虚拟内存？
- 如何提供应用程序需要的灵活性？
- 如何控制应用程序可以访问哪些内存位置，从而保证程序访问内存受到限制？
- 如何高效低完成这些要求？

**地址转换**

> 通过硬件支持的地址转换（address translation），将指令提供的**虚拟地址**翻译成数据在内存中的位置，即：**物理地址**。
>
> 硬件提供了底层的机制，为了实现虚拟内存，OS 必须管理内存：跟踪哪些内存位置是空闲的，哪些正在被使用；以及对内存使用方式的控制

**虚拟内存：base-and-bounds**

> CPU 上配有基址寄存器和界寄存器来帮助 OS 对地址进行动态重定位
>
> - 内存管理：需要为新的进程分配内存空间；回收已终止进程的内存空间；通过 free list 来管理内存
> - 基址/界寄存器管理：在进行上下文切换时，OS 需要重新设置这两个寄存器的值
> - 异常处理：当程序运行时出现异常，需要 OS 调用对应的代理处理它

## 模拟实验

**Q1**：在 seed 为 1,2,3 的情况下，运行 `./relocation.py`，是否产生的地址有效？

```bash
./relocation.py -s 1
# output
Virtual Address Trace
  VA  0: 0x0000030e (decimal:  782) --> SEGMENTATION VIOLATION
  VA  1: 0x00000105 (decimal:  261) --> VALID: 0x00003741 (decimal: 14145)
  VA  2: 0x000001fb (decimal:  507) --> SEGMENTATION VIOLATION
  VA  3: 0x000001cc (decimal:  460) --> SEGMENTATION VIOLATION
  VA  4: 0x0000029b (decimal:  667) --> SEGMENTATION VIOLATION

./relocation.py -s 2
./relocation.py -s 3
```



**Q2**：运行`./relocation.py -s 0 -n 10`，为了保证产生的虚拟地址有效，则 `-l` 的值至少应该为多少？

```bash
Virtual Address Trace
  VA  0: 0x000001ae (decimal:  430) --> PA or segmentation violation?
  VA  1: 0x00000109 (decimal:  265) --> PA or segmentation violation?
  VA  2: 0x0000020b (decimal:  523) --> PA or segmentation violation?
  VA  3: 0x0000019e (decimal:  414) --> PA or segmentation violation?
  VA  4: 0x00000322 (decimal:  802) --> PA or segmentation violation?
  VA  5: 0x00000136 (decimal:  310) --> PA or segmentation violation?
  VA  6: 0x000001e8 (decimal:  488) --> PA or segmentation violation?
  VA  7: 0x00000255 (decimal:  597) --> PA or segmentation violation?
  VA  8: 0x000003a1 (decimal:  929) --> PA or segmentation violation?
  VA  9: 0x00000204 (decimal:  516) --> PA or segmentation violation?
```

则 `-l` 的值至少应该为 930



**Q3**：运行`./relocation.py -s 0 -n 10 -l 100`，则 `base` 最大值可以为多少，使得它整个地址空间仍然可以适应物理内存？

> `Base` 的最大值可以为：`16k - 100`



**Q4**：使用更大的地址空间（`-a`）和物理内存（`-p`）测试

```bash
./relocation.py -a 32k -p 1024k -c       
# output
ARG seed 0
ARG address space size 32k
ARG phys mem size 1024k

Base-and-Bounds register information:

  Base   : 0x000c2094 (decimal 794772)
  Limit  : 15109

Virtual Address Trace
  VA  0: 0x000035d5 (decimal: 13781) --> VALID: 0x000c5669 (decimal: 808553)
  VA  1: 0x00002124 (decimal: 8484) --> VALID: 0x000c41b8 (decimal: 803256)
  VA  2: 0x00004171 (decimal: 16753) --> SEGMENTATION VIOLATION
  VA  3: 0x000033d4 (decimal: 13268) --> VALID: 0x000c5468 (decimal: 808040)
  VA  4: 0x00006453 (decimal: 25683) --> SEGMENTATION VIOLATION

# another
./relocation.py -a 10k -p 1024k -c
```



**Q5**：随机生成的虚拟地址在什么情况下才是有效的？

> 生成的虚拟地址（VA）必须小于界地址寄存器的值才是有效的





















