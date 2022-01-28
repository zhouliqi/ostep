# Paging: Faster Translations (TLBs)

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-tlbs.pdf)

**CRUX**

- 如何提高地址翻译的速度（因为页表的存在，VA 的翻译需要额外访问内存）？
- 需要什么硬件的支持？OS 需要做什么？
- 如何设计 TLB 的替换策略？

**TLB**

- 它是 MMU 的一部分，访问它的速度很快
- 缓存 VPN -> PFN 的映射关系
- 时间/空间局部性原理

```bash
# A TLB Entry
VPN | PFN | other bits
```

对于虚拟地址（VA），首先获取它的 VPN，然后在 TLB 中查找是否存在该映射

- 如果存在（TLB hit），那么直接获得对应的 PFN，就不需要再到页表中查找
- 如果不存在（TLB miss），那么需要去主存中查找对应到页表，获得该 VPN 所映射到 PFN，再把它缓存到 TLB 中

**处理 TLB Miss**

- 硬件管理的 TLBs（例如 x86）
- 软件管理的 TLBs（RISC），使用 Trap 机制处理

---

## 问题

**Q1**：`gettimeofday()` 的精度是多少？

> `gettimeofday()` 的精度是微秒，无法用来测量单次访问内存的时间；可以访问多次，然后求平均值



**Q2**：编写一个 `tlb.c` 的程序，用来统计访问每一页的开销，该程序的输入为需要访问页的个数以及实验的次数

```c
#include<stdio.h>
#include<sys/time.h>
#include<stdlib.h>

#define PAGESIZE 4096

int main(int argc, char *argv[]) {
    if (argc != 3) {
        perror("error on parameters");
        exit(1);
    }
    int NUMPAGES = atoi(argv[1]);
    int trials = atoi(argv[2]);
    int jump = PAGESIZE / sizeof(int);
    int *a = (int *)malloc(NUMPAGES * PAGESIZE / sizeof(int));

    struct timeval start, end;
    gettimeofday(&start, NULL);
    for (int k = 0; k < trials; ++k) {
        for (int i = 0; i < NUMPAGES; i += jump) {
            a[i] += 1;
        }
    }
    gettimeofday(&end, NULL);
    printf("%lf %d %d\n", (((double)end.tv_usec - start.tv_usec) / NUMPAGES) / trials, end.tv_usec, start.tv_usec);
    free(a);
	return 0;
}
```



**Q3**：编写一个简单的脚本，测量多次

```python
import os
i = 1
while i < 2000 :
	print('\nnumber of page: ' + str(i))
	val = os.system('./a.out ' + str(i) + ' ' + str(10000))
	i = i * 2
```

```bash
number of page: 1
NUMPAGES = 1, tries = 10000
0.002900 986686 986657

number of page: 2
NUMPAGES = 2, tries = 10000
0.002250 991021 990976

number of page: 4
NUMPAGES = 4, tries = 10000
0.002300 995303 995211

number of page: 8
NUMPAGES = 8, tries = 10000
0.002312 999632 999447

number of page: 16
NUMPAGES = 16, tries = 10000
0.005137 4489 3667

number of page: 32
NUMPAGES = 32, tries = 10000
0.009975 11716 8524

number of page: 64
NUMPAGES = 64, tries = 10000
0.010800 22764 15852

number of page: 128
NUMPAGES = 128, tries = 10000
0.010570 40756 27227

number of page: 256
NUMPAGES = 256, tries = 10000
0.009619 68977 44353

number of page: 512
NUMPAGES = 512, tries = 10000
0.009810 123416 73187

number of page: 1024
NUMPAGES = 1024, tries = 10000
0.009671 226587 127551
```



**Q4**：画出时间随页面数的变化图

```python
import numpy as np
import pandas as pd
import seaborn as sns
# sns.set_theme(style="darkgrid")

num = [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024]
times = [2.9, 2.25, 2.3, 2.312, 5.137, 9.975, 10.8, 10.57, 9.61, 9.81, 9.671]
datas = pd.DataFrame.from_dict(
    {
    'Time Per Access (ns)': pd.Series([2.9, 2.25, 2.3, 2.312, 5.137, 9.975, 10.8, 10.57, 9.61, 9.81, 9.671]),
    'num of pages': pd.Series([1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024])
    }
)
sns.lineplot(x="num of pages", y="Time Per Access (ns)", data=datas)
```



**Q5**：禁止编译器的优化，编译的时候添加命令 `-O0`



**Q6**：使用虚拟机限制只有一个CPU即可；如果代码移动到另一个CPU，那么显然TLB结构都是未命中，会增加访问时间。



**Q7**：在访问数组 `a` 前首先初始化它，会对结果有什么影响？

```c++
#include<stdio.h>
#include<sys/time.h>
#include<stdlib.h>

#define PAGESIZE 4096

int main(int argc, char *argv[]) {
    if (argc != 3) {
        perror("error on parameters");
        exit(1);
    }
    int NUMPAGES = atoi(argv[1]);
    int trials = atoi(argv[2]);
    int jump = PAGESIZE / sizeof(int);
    int *a = (int *)malloc(NUMPAGES * PAGESIZE * sizeof(int));
    printf("NUMPAGES = %d, tries = %d\n", NUMPAGES, trials);

    for (int i = 0; i < NUMPAGES * jump; i++) {
        a[i] = 0;
    }

    struct timeval start, end;
    gettimeofday(&start, NULL);
    for (int k = 0; k < trials; ++k) {
        for (int i = 0; i < NUMPAGES; i += jump) {
            a[i] += 1;
        }
    }
    gettimeofday(&end, NULL);
    printf("%lf %d %d\n", (((double)end.tv_usec - start.tv_usec) / NUMPAGES) / trials, end.tv_usec, start.tv_usec);
    free(a);
	return 0;
}
```















