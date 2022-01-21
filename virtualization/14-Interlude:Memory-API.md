# Interlude: Memory API

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/vm-api.pdf)

**内存的类型**

- Stack Memory（由编译器维护）
- Heap Memory（由编程人员维护）

```c
void func() {
    // 首先，由于声明了一个 int* 型的指针，编译器会为它在栈空间保留一定大小的空间
    // 然后，调用 malloc 会在堆中申请一块内存空间，成功返回这片空间的首地址，失败返回 NULL
    // 最后，将地址保存在第一步的栈空间中
    int *x = (int *) malloc(sizeof(int));
    ...
    free(X);
}
```

---

## 问题

**Q1**：编写一个有 bug 程序，不对 `int` 型指针初始化，然后解引用它，看看会出什么问题？

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
	int *ptr = NULL;
	printf("%d\n", *ptr);
	return 0;
}
```

```bash
gcc -o null null.c
./null
# output
[1]    43150 segmentation fault (core dumped)
```



**Q2**：使用 gdb 运行上面的程序，看看会输出什么？

> 如果要用 gdb 调试 c 程序，则编译时需要加上 `-g` 选项

```bash
gcc -g -o null null.c
gdb null
(gdb) run

# output
Program received signal SIGSEGV, Segmentation fault.
0x0000555555554665 in main (argc=1, argv=0x7fffffffc398) at null.c:6
6               printf("%d\n", *ptr);
```



**Q3**：最后，在这个程序上使用 `valgrind` 工具，看看会输出什么信息？

> 如果没有安装 `valgrind` 的话，先运行命令 `sudo apt update` 和 `sudo apt install valgrind`

```bash
valgrind --leak-check=yes ./null
==1788== Memcheck, a memory error detector
==1788== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==1788== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==1788== Command: ./null
==1788==
==1788== Invalid read of size 4
==1788==    at 0x108665: main (null.c:6)
==1788==  Address 0x0 is not stack'd, malloc'd or (recently) free'd
==1788==
==1788==
==1788== Process terminating with default action of signal 11 (SIGSEGV)
==1788==  Access not within mapped region at address 0x0
==1788==    at 0x108665: main (null.c:6)
==1788==  If you believe this happened as a result of a stack
==1788==  overflow in your program's main thread (unlikely but
==1788==  possible), you can try to increase the size of the
==1788==  main thread stack using the --main-stacksize= flag.
==1788==  The main thread stack size used in this run was 8388608.
==1788==
==1788== HEAP SUMMARY:
==1788==     in use at exit: 0 bytes in 0 blocks
==1788==   total heap usage: 0 allocs, 0 frees, 0 bytes allocated
==1788==
==1788== All heap blocks were freed -- no leaks are possible
==1788==
==1788== For counts of detected and suppressed errors, rerun with: -v
==1788== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
[1]    1788 segmentation fault (core dumped)  valgrind --leak-check=yes ./null
```



**Q4**：编写一个程序，它使用 `malloc` 申请了一块内存，但是在程序退出前并未释放这块内存。那么程序运行的时候会怎么样？能使用 gdb 发现问题吗？使用 `valgrind` 呢？

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
	int *ptr = (int *)malloc(sizeof(int));
	return 0;
}
```

```bash
gcc -g -o malloc_bug malloc_bug.c
./malloc_bug # no output

(gdb) run
[Inferior 1 (process 4172) exited normally]

valgrind --leak-check=yes ./malloc_bug
==4474== Memcheck, a memory error detector
==4474== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==4474== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==4474== Command: ./malloc_bug
==4474== 
==4474== 
==4474== HEAP SUMMARY:
==4474==     in use at exit: 4 bytes in 1 blocks
==4474==   total heap usage: 1 allocs, 0 frees, 4 bytes allocated
==4474== 
==4474== 4 bytes in 1 blocks are definitely lost in loss record 1 of 1
==4474==    at 0x4C31B0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==4474==    by 0x108662: main (malloc_bug.c:5)
==4474== 
==4474== LEAK SUMMARY:
==4474==    definitely lost: 4 bytes in 1 blocks
==4474==    indirectly lost: 0 bytes in 0 blocks
==4474==      possibly lost: 0 bytes in 0 blocks
==4474==    still reachable: 0 bytes in 0 blocks
==4474==         suppressed: 0 bytes in 0 blocks
==4474== 
==4474== For counts of detected and suppressed errors, rerun with: -v
==4474== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```



**Q5**：编写一个程序，它会使用 `malloc` 申请一个大小为 100 的 `int` 型数组，然后初始化这 100 个位置为 0，当你运行这个程序时会发生什么？当你使用 valgrind 运行是会发生什么？程序是否正确？

> 未释放申请的内存

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
	int n = 100;
	int *ptr = (int *)malloc(n * sizeof(int));
	ptr[100] = 0;
	free(ptr);
	return 0;
}
```

```bash
gcc -g -o malloc_bug malloc_bug.c
./malloc_bug # no output

valgrind --leak-check=yes ./malloc_bug
==10009== Memcheck, a memory error detector
==10009== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==10009== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==10009== Command: ./malloc_bug
==10009== 
==10009== Invalid write of size 4
==10009==    at 0x1086BF: main (malloc_bug.c:12)
==10009==  Address 0x522f1d0 is 0 bytes after a block of size 400 alloc'd
==10009==    at 0x4C31B0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==10009==    by 0x1086B0: main (malloc_bug.c:9)
==10009== 
==10009== 
==10009== HEAP SUMMARY:
==10009==     in use at exit: 0 bytes in 0 blocks
==10009==   total heap usage: 1 allocs, 1 frees, 400 bytes allocated
==10009== 
==10009== All heap blocks were freed -- no leaks are possible
==10009== 
==10009== For counts of detected and suppressed errors, rerun with: -v
==10009== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```



**Q6**：编写一个程序，，它会使用 `malloc` 申请一个大小为 100 的 `int` 型数组，再释放这块内存；然后再打印数组的某个位置的值。程序可以运行吗？当你使用 valgrind 运行是会发生什么？

> 可以，结果输出了数字的值；检测到了两个无效的读取

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
	int n = 100;
	int *ptr = (int *)malloc(n * sizeof(int));
	ptr[0] = 0;
	ptr[50] = 50;
	free(ptr);
	printf("%d %d\n", ptr[0], ptr[50]);
	return 0;
}
```

```bash
gcc -g -o malloc_bug malloc_bug.c
./malloc_bug
# output
0 50

valgrind --leak-check=yes ./malloc_bug
==10328== Memcheck, a memory error detector
==10328== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==10328== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==10328== Command: ./malloc_bug
==10328==
==10328== Invalid read of size 4
==10328==    at 0x108735: main (malloc_bug.c:19)
==10328==  Address 0x522f108 is 200 bytes inside a block of size 400 free'd
==10328==    at 0x4C32D3B: free (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==10328==    by 0x10872A: main (malloc_bug.c:18)
==10328==  Block was alloc'd at
==10328==    at 0x4C31B0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==10328==    by 0x108700: main (malloc_bug.c:9)
==10328==
==10328== Invalid read of size 4
==10328==    at 0x10873B: main (malloc_bug.c:19)
==10328==  Address 0x522f040 is 0 bytes inside a block of size 400 free'd
==10328==    at 0x4C32D3B: free (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==10328==    by 0x10872A: main (malloc_bug.c:18)
==10328==  Block was alloc'd at
==10328==    at 0x4C31B0F: malloc (in /usr/lib/valgrind/vgpreload_memcheck-amd64-linux.so)
==10328==    by 0x108700: main (malloc_bug.c:9)
==10328==
0 50
==10328==
==10328== HEAP SUMMARY:
==10328==     in use at exit: 0 bytes in 0 blocks
==10328==   total heap usage: 2 allocs, 2 frees, 1,424 bytes allocated
==10328==
==10328== All heap blocks were freed -- no leaks are possible
==10328==
==10328== For counts of detected and suppressed errors, rerun with: -v
==10328== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```



**Q7**：释放 Q6 中指向数组中间的指针值，此时会发生什么？需要使用其他工具发现问题吗？

> 不需要，程序会直接报错

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
	int n = 100;
	int *ptr = (int *)malloc(n * sizeof(int));
	free(ptr + 50);
	return 0;
}
```

```bash
gcc -g -o malloc_bug malloc_bug.c
./malloc_bug
# output
free(): invalid pointer
[1]    11450 abort (core dumped)  ./malloc_bug
```



**Q8**：使用其他的内存分配的接口，例如 `realloc` 来维护 `vector`

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
	int n = 1;
	int *p = (int *)malloc(n * sizeof(int));
	for (int i = 0; i < 100; i++) {
		if (i == n) {
			n <<= 2;
			p = (int *)realloc(p, n * sizeof(int));
		}
		p[i] = i;
	}
	free(p);
	return 0;
}
```



**Q9**：掌握 gdb 和 valgrind 的用法













