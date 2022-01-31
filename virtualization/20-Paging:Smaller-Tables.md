# Paging: Smaller Tables

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-smalltables.pdf)

**CRUX**

- 如何处理页表过大的问题？



**方法**

- 增大每个页面的大小，这样可以减少页表项的数目，但是可能会页面内部空间的浪费（内部碎片）
- 分页和分段相结合
- 多级页表

> 在多级页表中，TLB miss 产生的开销将大于单个页表（时间-空间的 trade-off）

---

## 模拟实验

**Q1**：如果是一个页表，你需要一个寄存器来记录这个页表在内存中的位置；如果现在是二级/三级页表，此时需要多少个寄存器？

> 一个



**Q2**：使用随机种子 0，1，2 来生成测试，每次翻译的时候需要访问多少次内存？

```bash
./paging-multilevel-translate.py -s 0

Virtual Address 611c (11000 01000 11100)

# page 108
83 fe e0 da 7f | d4 7f eb be 9e | d5 ad e4 ac 90 | d6 92 d8 c1 f8 | 9f e1 ed e9 a1 | e8 c7 c2 a9 d1 | db ff

# 0x611c 的高 5 位的值是 24，所以在页目录中的值为：0xa1(10100001)，最高位为有效位，则接着去 0x21(33) 找

# page 33
7f 7f 7f 7f 7f 7f 7f 7f b5 7f 9d 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f 7f f6 b1 7f 7f 7f 7f

# 0x611c 的中间 5 位的值是 8，所以在第 33 页中的值为：0xb5(10110101)，取出物理页号，加上偏移量得到：0x6bc(011010111100)
# 最后去物理页 53 得到值：08
```

```bash
./paging-multilevel-translate.py -s 1
./paging-multilevel-translate.py -s 2
```

> 如果该 VPN -> PFN 在 TLB 中，则访存一次；否则访存 3 次



**Q3**：由于时间局部性和空间局部性原理，将最近或近期访问的页表项保存着缓存中。如果命中，可以减少额外访问内存的次数，从而提升性能













