---
title : C++11学习笔记-----线程库std::thread
date : 2018-02-06
categories : C++
tags : C++
---

在以前，要想在C++程序中使用线程，需要调用操作系统提供的线程库，比如linux下的&lt;pthread.h&gt;。但毕竟是底层的C函数库，没有什么抽象封装可言，仅仅透露着一种简单，暴力美

C++11在语言级别上提供了线程的支持，不考虑性能的情况下可以完全代替操作系统的线程库，而且使用起来非常方便，为开发程序提供了很大的便利

<!--more-->

# Linux下的原生线程库

## pthread库函数

创建线程采用pthread_create函数，函数声明如下

```c
#include <pthread.h>
/* 
 * pid : 线程id，传入pthread_t类型的指针，函数返回时会返回线程id
 * attr : 线程属性
 * func : 线程调用的函数
 * arg : 给函数传入的参数
 */
int pthread_create(pthread_t* pid, const pthread_attr_t* attr, void*(*func)(void*), void* arg);
```

可以发现，创建线程时只能传递一个参数给线程函数，所以如果想要给函数传入多个参数的话就需要动点歪脑筋，比如如果是类对象的话可以传入this指针，或者也可以将参数封装成一个struct传进去。不过总感觉不太优雅

当一个进程创建一个线程时，虽然线程运行在主进程的内存空间中，但是每个线程也有自己的私有空间(资源，如局部变量等)。当程序正常退出时，执行者是希望线程的私有资源可以成功被操作系统回收(即资源回收)，这就需要主进程在退出之前显示调用pthread_join函数，该函数会等待参数id代表的线程退出，然后回收其资源，如果调用时目标线程还没有运行结束，那么调用方(主进程)会被阻塞

当然，如果觉得这样太麻烦，也可以使用pthread_detach函数主动分离线程，这样，当线程运行结束后会由操作系统自动回收资源，不再需要主进程操心

还有一个常用的api是pthread_exit，用于主动结束当前线程

## 示例：若干线程并发对一个数进行自增操作

使用示例，创建10个线程对进程全局变量n做加法，每个线程加10000次

```c
#include <unistd.h>
#include <pthread.h>
#include <sys/types.h>

#include <iostream>

long long int n = 0;
void *thread_func(void* arg)
{
    for(int i = 0; i < 10000; ++i)
        ++n;
    ::pthread_exit(nullptr);
}

int main()
{
    for(int i = 0; i < 10; ++i)
    {
        pthread_t pid;
        ::pthread_create(&pid, nullptr, thread_func, nullptr); 
        ::pthread_detach(pid);
    }
    ::sleep(1);	//等待所有线程正常退出
    std::cout << n << std::endl;
    return 0;
}
```

当然最后这个结果绝不可能是100000，要想保证正确性，需要互斥锁协助。

# C++11线程库

简单介绍了posix原生线程的使用，一方面用于复习，另一方面自然是为了引出主角。C++11引入线程库std::thread，使得C++在语言级别上支持线程，虽然大家都说性能不咋地，但是用起来自然是方便许多。突出的几个特点有

* 支持lambda，创建线程可以传入lambda作为执行函数，太方便了有木有~
* 支持任意多个参数，由于C++模板支持可变参数列表，所以实现多参数传递还是蛮容易的
* 使用方便，各种函数都经过了良好设计，使用起来比posix不知道高到哪里去了

> 小插曲，介绍了这么多好处当然也要吐槽一下，编译C++11线程库居然要手动链接-lpthread库....

## 创建线程的几种方式

使用线程库需要引入头文件&lt;thread&gt;，有下面几种方法创建线程

```c
#include <iostream>
#include <thread>
#include <chrono>
#include <functional>


class ThreadTask
{
public:
    ThreadTask(int a, int b)
        : a_(a), b_(b)
    {  }

    void operator()()
    {
        std::cout << "hello " << std::this_thread::get_id() << std::endl;
        std::cout << "a + b = " << a_ + b_ << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << "world " << std::this_thread::get_id() << std::endl;
    }

private:
    int a_;
    int b_;
};

void func(int a)
{
    std::cout << "hello " << std::this_thread::get_id() << std::endl;
  	std::cout << a << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "world " << std::this_thread::get_id() << std::endl;
}

int main()
{
    std::thread t1(func, 1);	//接收一个函数指针和参数列表
    std::thread t2([]() {
        std::cout << "hello " << std::this_thread::get_id() << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << "world " << std::this_thread::get_id() << std::endl;
    });							//接收lambda
    ThreadTask task(1, 2); 
    std::thread t3(task);		//接收函数对象
    
    t1.join(); //t1.detach();
    t2.join(); //t2.detach();
    t3.join(); //t3.detach();
               //std::this_thread::sleep_for(std::chrono::seconds(1));
    return 0;
}
```

> 需要注意的是，std::thread对象是不允许拷贝的，拷贝构造函数和拷贝赋值运算符被指定为delete

## 线程的移动语义

虽然不允许拷贝，但是std::thread是允许移动的

```c
int main()
{
    std::thread t1([]() {
        std::cout << "hello " << std::this_thread::get_id() << std::endl;
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << "world " << std::this_thread::get_id() << std::endl;
    });
    std::thread t2(std::move(t4));	//移动构造函数
    std::thread t3;
    t3 = std::move(t2);	//移动赋值运算符

    t3.join();
    return 0;
}
```

## 示例：利用std::thread实现并行的accumulate函数

标准库std::accumulate函数用于对给定区间的元素依次运算，默认是加法，使用示例

```c
#include <iostream>
#include <algorithm>
#include <vector>

using namespace std;

int main()
{
    vector<int> v{1, 3, 5, 7, 9, 12};
 	/* 输出37,即所有元素的和 */
    std::cout << std::accumulate(v.begin(), v.end(), 0) << std::endl;
    /* 输出5，即所有元素异或的结果 */
    std::cout << std::accumulate(v.begin(), v.end(), 0, bit_xor<int>()) << std::endl;
    return 0;
}
```



下面利用std::thread实现并行accumulate，即利用多线程的优势同时计算不同区块

首先是根据给定区间计算创建的线程个数，std::thread标准库中提供了hardware_concurrency()函数，该函数返回当前计算机支持的并发线程数，通常是cpu核数，如果值无法计算则返回0。

在此之前，最好规定每个线程最小计算的元素个数，不然如果创建线程过多，导致每个线程计算的元素个数很少，那么创建线程带来的开销就会大于运算开销，得不偿失。所以可以规定每个线程最少计算20个元素

```c
template <class Distance>
auto getThreadNums(Distance count)
{
    auto avaThreadNums = std::thread::hardware_concurrency();
    auto minCalNums = 20;
  	/* 将count向上取整到20的整数倍，计算最大需要多少个线程 */
    auto maxThreadNums = ((count + (minCalNums - 1)) & (~(minCalNums - 1))) / minCalNums;
    /* 选择二者中最合适的那个 */
    return avaThreadNums == 0 ? maxThreadNums : std::min(static_cast<int>(avaThreadNums), static_cast<int>(maxThreadNums));
}
```

通过线程个数，就可以计算每个小区间负责的元素个数，从而将区间[front, last)拆分成若干个小区间

```c
auto count = std::distance(first, last);
int threadNums = getThreadNums(count);
int blockSize = count / threadNums;	//区间大小
```

接下来的工作就是创建threadNums个线程，每个线程调用std::accumulate计算自己负责的区间结果，同时将结果保存，最后将每个区间的结果再求一次std::accumulate

```c
template <class InputIt, class T>
T parallel_accumulate(InputIt first, InputIt last, T init)
{
    auto count = std::distance(first, last);
    int threadNums = getThreadNums(count);
    int blockSize = count / threadNums;
    std::vector<std::thread> threads;
    std::vector<T> results(threadNums);
    auto front = first;
    for(int i = 0; i < threadNums; ++i)
    {
        auto back = front;
        if(i != threadNums - 1)
            std::advance(back, blockSize);
        else
            back = last;
        threads.emplace_back([front, back, &results, i, init] { results[i] = std::accumulate(front, back, init); });
        front = back;
    }
    for(auto& th : threads)
        th.join();
    return std::accumulate(results.begin(), results.end(), init);
}
```

测试代码为

```c
#include <iostream>
#include <algorithm>
#include <vector>
#include <thread>
#include <future>
#include <chrono>
#include <functional>
#include <random>

#include "../../tinySTL/Profiler/profiler.h"


/* parallel_accumulate的实现 */
...

int main()
{
    std::vector<long long int> v(100000000);
    std::random_device rd;
    std::generate(v.begin(), v.end(), [&rd]() { return rd() % 1000; });
    tinystl::Profiler::ProfilerInstance::start(); 
    auto result = parallel_accumulate(v.begin(), v.end(), 0);
    tinystl::Profiler::ProfilerInstance::finish(); 
    tinystl::Profiler::ProfilerInstance::dumpDuringTime(); 
    std::cout << result << std::endl;

    tinystl::Profiler::ProfilerInstance::start(); 
    result =  std::accumulate(v.begin(), v.end(), 0);
    tinystl::Profiler::ProfilerInstance::finish(); 
    tinystl::Profiler::ProfilerInstance::dumpDuringTime(); 
    std::cout << result << std::endl;
    return 0;
}
```

输出结果

```c
g++ thread.cpp -o thread -std=c++14 -lpthread -g	//编译
./thread 	//执行
total 213.717 milliseconds
-1584776879
total 776.649 milliseconds
-1584776879
```

虽然都溢出了，但是可以看出并行计算快很多



# 小结

C++11提供的线程库使用起来比较方便，以前在学习多线程编程时一直使用的posix原生线程库，刚刚接触C++11时感觉方便很多，后面会继续学习互斥锁，条件变量的使用。