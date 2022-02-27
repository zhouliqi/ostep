# Lock-based Concurrent Data Structures

> [讲义链接](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-locks-usage.pdf)

## 问题

**Q1**：`gettimeofday()` 函数的精度如何？它可以测量的最小时间间隔是多少？

> 它的精度可以达到微妙

```c++
#include<sys/time.h>

struct  timeval {
    long  tv_sec;  // 秒
    long  tv_usec; // 微妙
}；

int gettimeofday(struct  timeval*tv, struct  timezone *tz);
```



**Q2**：编写一个简单的并发 `count`，不断地增加 `count` 的值；并且随着线程个数的增加，它的运行时间如何变化？

```c++
#include<sys/time.h>
#include<thread>
#include<mutex>
#include <vector>
#include<iostream>

const size_t N = 1e8;
const size_t CORES = 10;

class Count {
private:
	size_t count_;
	std::mutex mu_;
public:
	explicit Count() : count_(0) {}
	void inc() {
		std::lock_guard<std::mutex> guard(mu_);
		count_ += 1;
	}
	inline size_t get() {
		return count_;
	}
};

Count c;

void worker() {
	for (size_t i = 0; i < N / CORES; ++i) {
		c.inc();
	}
}

int main(int argc, char const *argv[]) {
	struct timeval start, end;
	gettimeofday(&start, NULL);
	std::vector<std::thread> tasks;
	for (size_t i = 0; i < CORES; ++i) {
		tasks.emplace_back(std::thread{worker});
		tasks.back().join();
	}
	gettimeofday(&end, NULL);
	printf("final count = %ld, total time = %ld\n", c.get(), end.tv_sec - start.tv_sec);
	return 0;
}
```
