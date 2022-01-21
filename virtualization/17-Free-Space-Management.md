# Free-Space Management

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-freespace.pdf)

- 如何处理外部碎片？

> 例如：`|0|--- free ---|10|--- used ---|20|--- free ---|30| `，如果此时程序请求 `15` 字节的空间，则该请求失败，因为没有一个长度大于等于 15 字节的连续内存空间，然而很明显，此时空闲空间总数为 20 字节

**对于变长内存请求**

- 如何管理空闲的内存？
- 什么策略可以最小化外部碎片？
- 它的时间、空间开销如何？

---

## 仿真实验

**Q1**：使用标志 `-n 10 -H 0 -p BEST -s 0` 生成测试，`alloc()/free()` 的返回值是什么？在每次请求之后，free list 的状态是什么？

```bash
./malloc.py -n 10 -H 0 -p BEST -s 0

# output
ptr[0] = Alloc(3) returned 1000 (searched 1 elements)
Free List [Size 1]: [ addr:1003 sz:97 ]

Free(ptr[0])
returned 0
Free List [ Size 2 ]: [ addr:1000 sz:3 ][ addr:1003 sz:97 ]

ptr[1] = Alloc(5) returned 1003 (searched 2 elements)
Free List [ Size 2 ]: [ addr:1000 sz:3 ][ addr:1008 sz:92 ]

Free(ptr[1])
returned 0
Free List [Size 3]: [ addr:1000 sz:3 ][ addr:1003 sz:5 ][ addr:1008 sz:92 ]

ptr[2] = Alloc(8) returned 1008 (searched 3 elements)
Free List [Size 3]: [ addr:1000 sz:3 ][ addr:1003 sz:5 ][ addr:1016 sz:84 ]

Free(ptr[2])
returned 0
Free List [Size 4]: [ addr:1000 sz:3 ][ addr:1003 sz:5 ][ addr:1008 sz:8 ][ addr:1016 sz:84 ]

ptr[3] = Alloc(8) returned 1008 (searched 4 elements)
Free List [Size 3]: [ addr:1000 sz:3 ][ addr:1003 sz:5 ][ addr:1016 sz:84 ]

Free(ptr[3])
returned 0
Free List [Size 4]: [ addr:1000 sz:3 ][ addr:1003 sz:5 ][ addr:1008 sz:8 ][ addr:1016 sz:84 ]

ptr[4] = Alloc(2) returned 1000 (searched 4 elements)
Free List [Size 4]: [ addr:1002 sz:1 ][ addr:1003 sz:5 ][ addr:1008 sz:8 ][ addr:1016 sz:84 ]

ptr[5] = Alloc(7) returned 1008 (searched 4 elements)
Free List [Size 4]: [ addr:1002 sz:1 ][ addr:1003 sz:5 ][ addr:1015 sz:1 ][ addr:1016 sz:84 ]
```



**Q2**：在 Q1 的基础上，如果使用 WORST 适应策略，那么结果又是如何？

> 因为每次找最大的匹配，又不会合并，所以容易产生更多的碎片

```bash
./malloc.py -n 10 -H 0 -p WORST -s 0

# output
ptr[0] = Alloc(3) returned 1000 (searched 1 elements)
Free List [Size 1] : [ addr:1003 sz:97 ]

Free(ptr[0])
returned 0
Free List [Size 2] : [ addr:1000 sz:3 ] [ addr:1003 sz:97 ]

ptr[1] = Alloc(5) returned 1003 (searched 2 elements)
Free List [Size 2] : [ addr:1000 sz:3 ] [ addr:1008 sz:92 ]

Free(ptr[1])
returned 0
Free List [Size 3] : [ addr:1000 sz:3 ] [ addr:1003 sz:5 ] [ addr:1008 sz:92 ]

ptr[2] = Alloc(8) returned 1008 (searched 3 elements)
Free List [Size 3] : [ addr:1000 sz:3 ] [ addr:1003 sz:5 ] [ addr:1016 sz:84 ]

Free(ptr[2])
returned 0
Free List [Size 4] : [addr:1000 sz:3] [addr:1003 sz:5] [addr:1008 sz:8] [addr:1016 sz:84]

ptr[3] = Alloc(8) returned 1016 (searched 4 elements)
Free List [Size 4] : [addr:1000 sz:3] [addr:1003 sz:5] [addr:1008 sz:8] [addr:1024 sz:76]

Free(ptr[3])
returned 0
Free List [Size 5] : [addr:1000 sz:3] [addr:1003 sz:5] [addr:1008 sz:8] [addr:1016 sz:8] [addr:1024 sz:76]

ptr[4] = Alloc(2) returned 1024 (searched 5 elements)
Free List [Size 5] : [addr:1000 sz:3] [addr:1003 sz:5] [addr:1008 sz:8] [addr:1016 sz:8] [addr:1026 sz:74]

ptr[5] = Alloc(7) returned 1033 (searched 5 elements)
Free List [Size 5] : [addr:1000 sz:3] [addr:1003 sz:5] [addr:1008 sz:8] [addr:1016 sz:8] [addr:1033 sz:67]
```



**Q3**：在 Q1 的基础上，如果使用 FIRST 适应策略，那么结果又是如何？速度又怎么样？

```bash
./malloc.py -n 10 -H 0 -p FIRST -s 0

# output
ptr[0] = Alloc(3) returned 1000 (searched 1 elements)
Free List [Size 1] : [ addr:1003 sz:97 ]

Free(ptr[0])
returned 0
Free List [Size 2] : [ addr:1000 sz:3 ] [ addr:1003 sz:97 ]

ptr[1] = Alloc(5) returned 1003 (searched 2 elements)
Free List [Size 2] : [ addr:1000 sz:3 ] [ addr:1008 sz:92 ]

Free(ptr[1])
returned 0
Free List [Size 3] : [ addr:1000 sz:3 ] [ addr:1003 sz:5 ] [ addr:1008 sz:92 ]

ptr[2] = Alloc(8) returned 1008 (searched 3 elements)
Free List [Size 3] : [ addr:1000 sz:3 ] [ addr:1003 sz:5 ] [ addr:1016 sz:84 ]

Free(ptr[2])
returned 0
Free List [Size 4] : [addr:1000 sz:3] [addr:1003 sz:5] [addr:1008 sz:8] [addr:1016 sz:84]

ptr[3] = Alloc(8) returned 1008 (searched 3 elements)
Free List [Size 3] : [ addr:1000 sz:3 ] [ addr:1003 sz:5 ] [ addr:1016 sz:84 ]

Free(ptr[3])
returned 0
Free List [Size 4] : [addr:1000 sz:3] [addr:1003 sz:5] [addr:1008 sz:8] [addr:1016 sz:84]

ptr[4] = Alloc(2) returned 1000 (searched 1 elements)
Free List [Size 4] : [addr:1002 sz:1] [addr:1003 sz:5] [addr:1008 sz:8] [addr:1016 sz:84]

ptr[5] = Alloc(7) returned 1008 (searched 3 elements)
Free List [ Size 4 ]: [addr:1002 sz:1] [addr:1003 sz:5] [addr:1015 sz:1] [addr:1016 sz:84]
```



**Q4**：对于上面三个问题，如果使用不同的排序方式（`-l ADDRSORT, -l SIZESORT+, -l SIZESORT-`），结果又会是怎么样？

```bash
./malloc.py -n 10 -H 0 -p BEST -s 0 -l ADDRSORT -c
./malloc.py -n 10 -H 0 -p BEST -s 0 -l SIZESORT+ -c
./malloc.py -n 10 -H 0 -p BEST -s 0 -l SIZESORT- -c

./malloc.py -n 10 -H 0 -p WORST -s 0 -l ADDRSORT -c
./malloc.py -n 10 -H 0 -p WORST -s 0 -l SIZESORT+ -c
./malloc.py -n 10 -H 0 -p WORST -s 0 -l SIZESORT- -c

./malloc.py -n 10 -H 0 -p FIRST -s 0 -l ADDRSORT -c
./malloc.py -n 10 -H 0 -p FIRST -s 0 -l SIZESORT+ -c
./malloc.py -n 10 -H 0 -p FIRST -s 0 -l SIZESORT- -c
```



**Q5**：合并 free list 也是挺重要的，现在使用 `-n 1000` 增加生成的请求，则 `request` 的次数会怎么变？分别运行：无合并；有合并的测试

> 可以看到合并 free list 后，每次搜索的次数明显减少

```bash
./malloc.py -n 1000 -H 0 -p BEST -s 0 -c | rg "searched"
./malloc.py -n 1000 -H 0 -p BEST -s 0 -C -c | rg "searched"
```



**Q6**：如果将 `-P` 设置为大于 50 会发生什么？如果分配内存的请求率接近 100 会发生什么？接近 0 会发生什么？

> 分配内存的请求率越接近 100，空闲的内存空间越少，则失败的可能性越高（无内存可用） 

```bash
./malloc.py -n 10 -H 0 -P 70 -c
./malloc.py -n 10 -H 0 -P 90 -c
./malloc.py -n 10 -H 0 -P 10 -c
```



**Q7**：什么样的请求可以产生一个有很高的碎片的内存空间，使用 `-A` 来创建一个这样的请求

> 不断分配小的内存区域

```bash
./malloc.py -H 0 -A +1,+1,+1,+1,+1,+1,...,+1
```



