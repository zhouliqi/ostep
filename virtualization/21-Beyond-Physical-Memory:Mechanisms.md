# Beyond Physical Memory: Mechanisms

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-beyondphys.pdf)

**CRUX**

- OS 如何利用更大、慢的存储设备来提供一个很大的虚拟地址空间的假象？



当程序从内存中取数据时会发生什么？

> 首先根据虚拟地址 VA 获得 VPN（Virtual Page Number），然后现在 TLB 中查找这个 VPN 是否存在。
>
> - 存在（TLB Hit）
> 	- 可以访问，那么通过得到的 PFN 和 OFFSET 构成对应的物理地址，然后访问该内存地址
> 	- 违反了保护位，则抛出 `PROTECTION_FAULT` 异常
> - 不存在（TLB Miss）
> 	- 从 PTBR 中得到页表在内存中的地址，然后根据 VPN 得到对应的 PTE（Page Table Entry）。如果该页表项无效，则抛出 `PROTECTION_FAULT` 异常；如果 `PTE.Present == false`，则抛出 `PAGE_FAULT` 异常；否则，将该项插入到 TLB 中，然后重新执行该指令

处理 **Page Fault**

```c++
PFN = FindFreePhysicalPage()
if (PFN == -1) 		  		// no free page found
	PFN = EvictPage()		// run replacement algorithm
DiskRead(PTE.DiskAddr, PFN) // sleep (waiting for I/O)
PTE.present = True 			// update page table with present
PTE.PFN = PFN 				// bit and translation (PFN)
RetryInstruction() 			// retry instruction
```

---

## 模拟实验

**Q1**：命令 `vmstat` 的使用（Linux 系统），并且运行 `mem.c`，查看程序运行时的变化

```bash
# in one window
vmstat 1

# in another window
./mem 1
```



**Q2**：使用 `./mem 1024` 和 `vmstat` 来查看程序运行时和结束后的 free 列的变化

```bash
# in one window
vmstat 1

# in another window
./mem 1024
```

可以看到当程序运行时，free 列的数值减少了很多；当程序退出后，又恢复到之前到数值



**Q3**：使用大的数值传给 `./mem`，然后查看 si 和 so 列是否为 0

```bash
# in one window
vmstat 1

# in another window
./mem 8000
```





**Q4**：当程序运行时 block I/O 这两列会怎么变？

> 当程序运行时，进入第一次循环，`bo` 会变大



**Q5**：性能测试

```bash
# in one window
vmstat 1

# in another window
./mem 5000

# output
loop 0 in 3430.02ms (bandwidth: 1457.72 MB/s)
```

```
# in one window
vmstat 1

# in another window
./mem 111616

# output
loop 0 in 79627.78ms (bandwidth: 1401.72 MB/s)
```



**Q6**：使用 `swapon -s` 查看交换区的空间，如果程序 `mem.c` 使用一个超过内存+交换区的值会发生什么？

> 当程序的内存超过内存+交换区时，分配内存失败



**Q7**：

















