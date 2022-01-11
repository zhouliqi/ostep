# Interlude: Process API

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/cpu-api.pdf)

## Questions

1. 了解 `fork` 父进程的时候 OS 做了那些工作？

> fork 系统调用的时候，子进程是父进程的拷贝（COW），父子进程都有自己的内存空间，所以在父进程修改 `x` 的值不会影响子进程；反之亦然。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
	printf("hello world (pid:%d)\n", (int) getpid());
	int x = 100;
	int rc = fork();
	if (rc < 0) {
		printf("Error on create process, exit!");
		exit(1);
	} else if (rc == 0) {
		x = 90;
		printf("Child pid = %d, and `x` value of child = %d\n", getpid(), x);
	} else {
		x = 110;
		printf("Parent pid = %d, and `x` value of Parent = %d\n", getpid(), x);
	}
	return 0;
}
```



2. 父进程 `open` 一个文件，`fork` 一个子进程，然后父子进程同时对这个文件读写会发生什么？

> 同上，`fork` 的时候子进程也拷贝了父进程打开的文件描述符，我这里测试的时候父子进程会接替执行

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>

int main(int argc, char *argv[]) {
	printf("hello world (pid:%d)\n", (int) getpid());
	int fd = open("./q2.txt", O_CREAT|O_WRONLY|O_TRUNC, S_IRWXU);
	char buf;
	if (fd == -1) {
		printf("Error on open a file, exit!");
		exit(1);
	}
	int rc = fork();
	// 子进程和父进程同时向打开的文件写入内容
	if (rc < 0) {
		printf("Error on create process, exit!");
		exit(1);
	} else if (rc == 0) {
		for (int i = 0; i < 100; i++) {
			buf = 'b';
			if (write(fd, &buf, 1) != 1) {
				printf("Error on write into file in child process, exit!");
				exit(1);
			}
		}
	} else {
		for (int i = 0; i < 100; i++) {
			buf = 'a';
			if (write(fd, &buf, 1) != 1) {
				printf("Error on write into file in parent process, exit!");
				exit(1);
			}
		}
	}
	return 0;
}
```



3. 不使用 `wait` 系统调用，让 `fork` 的子进程比父进程先运行

> 在父进程 `sleep` 一下，或者使用 `vfork` 系统调用

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char *argv[]) {
	int rc = fork();
	if (rc < 0) {
		printf("Error on create process, exit!");
		exit(1);
	} else if (rc == 0) {
		printf("hello\n");
	} else {
		sleep(1);
		printf("goodbay\n");
	}
	return 0;
}
```



4. 测试 `exec` 函数族

> `exec` 函数族是为了适应不同的调用形式和环境

```c
#define _GNU_SOURCE
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/wait.h>

#define MAX 6

int main(int argc, char *argv[]) {
	char * s = "/bin/ls";
	char * ss = "ls";
	char * home = "/";
	char * sv[] = {ss, home, NULL};
	for (int i = 0; i < MAX; ++i) {
		int rc = fork();
		if (rc < 0) {
			fprintf(stderr, "fork failed\n");
			exit(1);
		} else if (rc == 0) {
			// exec函数族分别是：execl, execle, execlp, execv, execvp, execvpe
			switch (i) {
			case 0:
				execl(s, ss, home, NULL);
				break;
			case 1:
				execle(s, ss, NULL, home);
				break;
			case 2:
				execlp(s, ss, home, NULL);
				break;
			case 3:
				execv(s, sv);
				break;
			case 4:
				execvp(ss, sv);
				break;
			case 5:
				// execvpe(ss, sv);
				break;
			default:
				break;
			}
		} else {
			wait(NULL);
		}
	}
	return 0;
}
```



5. `wait` 系统调用

> 父进程使用 `wait()` 返回子进程 `PID`，子进程由于本身没有子进程，所以返回 `-1`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
	int rc = fork();
	if (rc < 0) {
		fprintf(stderr, "fork failed\n");
		exit(1);
	} else if (rc == 0) {
		printf("hello, I am child (pid:%d)\n", (int) getpid());
		wait(NULL);
	} else {
		int rc_wait = wait(NULL); // wait 返回子进程的 PID
		printf("hello, I am parent (pid:%d), rc_wait = %d\n", (int) getpid(), rc_wait);
	}
	return 0;
}
```



6. `waitpid` 系统调用

> 在进程有子进程的时候有用

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
	pid_t cpid, w;
	int status;

	cpid = fork();
	if (cpid == -1) {
		perror("fork");
		exit(EXIT_FAILURE);
	} else if (cpid == 0) {
		printf("Child PID is %ld\n", (long) getpid());
	} else {
		w = waitpid(cpid, &status, WUNTRACED | WCONTINUED);
        if (w == -1) {
            perror("waitpid");
            exit(EXIT_FAILURE);
        }

       if (WIFEXITED(status)) {
            printf("exited, status=%d\n", WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("killed by signal %d\n", WTERMSIG(status));
        } else if (WIFSTOPPED(status)) {
            printf("stopped by signal %d\n", WSTOPSIG(status));
        } else if (WIFCONTINUED(status)) {
            printf("continued\n");
        }
	}
	return 0;
}
```



7. `close` 系统调用

> 子进程关闭了 `STDOUT_FILENO` 之后，那么使用 `printf` 就无法向屏幕显示内容了

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
	pid_t cpid;

	cpid = fork();
	if (cpid == -1) {
		perror("fork");
		exit(EXIT_FAILURE);
	} else if (cpid == 0) {
		printf("Before close STDOUT\n");
		close(STDOUT_FILENO);
		printf("After close STDOUT\n");
	} else {
		int rc_wait = wait(NULL); // wait 返回子进程的 PID
		printf("hello, I am parent (pid:%d), rc_wait = %d\n", (int) getpid(), rc_wait);
	}
	return 0;
}
```



8. `pipe` 系统调用

> 创建一个管道，然后再 `fork` 两个子进程，其中一个向管道的一端写内容，另一个向管道的另一端读取内容

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

#define MAX 256

int main(int argc, char *argv[]) {
	pid_t cpid[2];
	// fd[1] refers to the write end of the pipe
	// fd[0] refers to the read end of the pipe
	int fd[2];
	int p = pipe(fd);
	if (p < 0) {
		perror("pipe");
		exit(1);
	}

	int i = 0;
	char buf[MAX];
	for (int i = 0; i < 2; i++) {
		cpid[i] = fork();
		if (cpid[i] == -1) {
			perror("fork");
			exit(EXIT_FAILURE);
		} else if (cpid[i] == 0) {
			switch(i) {
			case 0:
				dup2(fd[1], STDOUT_FILENO);
				write(STDOUT_FILENO, "Hello, child process 1", MAX);
				break;
			case 1:
				dup2(fd[0], STDIN_FILENO);
				read(STDIN_FILENO, buf, MAX);
				printf("child read: %s\n", buf);
				return 2;
			}
			break;
		} else {
			waitpid(cpid[0], NULL, 0);
			waitpid(cpid[1], NULL, 0);
			printf("Finish, I am parent (pid:%d)\n", (int) getpid());
		}
    }

	return 0;
}
```

