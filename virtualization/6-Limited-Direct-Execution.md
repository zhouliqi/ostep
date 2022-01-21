# Mechanism: Limited Direct Execution

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-mechanisms.pdf)

> 这一节主要介绍 CPU 的上下文切换过程

---

## 问题

**Q1**：测试系统调用所花费的时间？

> 不断地调用 `read` 系统调用，使用 `gettimeofday` 函数来获取此刻的时间，计算调用前和调用后的差值即可

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <sys/time.h>
#include <time.h>

#define MAX 1000000

float time_diff(struct timeval *start, struct timeval *end) {
    return (end->tv_sec - start->tv_sec) + 1e-6 * (end->tv_usec - start->tv_usec);
}

void test_sys_call_cost() {
	char buf;
	int fd = open("./tmp.txt", O_CREAT|O_WRONLY|O_TRUNC, S_IRWXU);
	for (int i = 0; i < MAX; i++) {
		read(fd, &buf, 0);
	}
}

int main(int argc, char *argv[]) {
	struct timeval start;
    struct timeval end;

    gettimeofday(&start, NULL);
    test_sys_call_cost();
    gettimeofday(&end, NULL);
    printf("call %d read(), total cost: %0.8fs\n", MAX, time_diff(&start, &end));
    printf("average cost: %0.8fms\n", 1000 * time_diff(&start, &end) / MAX);

	return 0;
}
```



**Q2**：测试上下文切换所花费的时间

> 创建两个管道，在 `fork` 一个线程，不断地让父子进程通信

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <sys/time.h>
#include <time.h>

#define MAX 10
#define BUF_MAX 16

int main(int argc, char *argv[]) {
	int fd1[2], fd2[2];
	char buf1[BUF_MAX], buf2[BUF_MAX];
	struct timeval start;
    struct timeval end;

	if (pipe(fd1) < 0) {
		perror("pipe");
		exit(1);
	}
	if (pipe(fd2) < 0) {
		perror("pipe");
		exit(1);
	}
	int rc = fork();
	for (int i = 0; i < MAX; i++) {
		if (rc < 0) {
			perror("fork\n");
			exit(1);
		} else if (rc == 0) {
			read(fd1[0], buf1, BUF_MAX);
			gettimeofday(&end, NULL);
			printf("cost time: %0.4fms\n", 1.0 * (end.tv_usec - atol(buf1)) / 1000);
			gettimeofday(&start, NULL);
			sprintf(buf2, "%ld", start.tv_usec);
			write(fd2[1], buf2, BUF_MAX);
		} else {
			gettimeofday(&start, NULL);
			sprintf(buf1, "%ld", start.tv_usec);
			write(fd1[1], buf1, BUF_MAX);
			read(fd2[0], buf2, BUF_MAX);
			gettimeofday(&end, NULL);
			printf("cost time: %0.4fms\n", 1.0 * (end.tv_usec - atol(buf2)) / 1000);
		}
	}
}
```

