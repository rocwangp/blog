---
title : Redis源码剖析（二）io多路复用函数及事件驱动流程
date : 2018-01-04 13:19
categories : Redis
tags : Redis
---



作为服务器监听客户端请求的方法，io多路复用起到了不可忽略的作用，利用io复用监听的方法叫Reactor模式，在前一篇也提到过，使用io复用是现在常用的提高并发性的方法，而且效果显著。

通常io多路复用连同事件回调是一起出现的，在将文件描述符(套接字)注册到io多路复用函数中时，同时也需要保存当这个文件描述符被激活时调用的函数(称作回调函数)，这样，使用者无需考虑何时事件被激活又何时调用相应处理函数，只管注册即可，执行回调函数的任务由Reactor接管，极大提高了并发性

<!--more-->

在C语言中，回调函数通常是以函数指针的形式出现的(参考libevent)

在C++语言中，回调函数可以是函数指针，但是通常会是通过std::bind绑定的std::function对象，当然随着C++11的出现，也可以以lambda代替std::bind

既然Redis是C语言实现的，就老老实实使用函数指针好了，不过在此之前，先简单复习一下io多路复用函数

# io多路复用函数

## Linux平台三种io多路复用函数的区别

在不同的平台(linux，window)，存在着不同的io复用函数，以Linux平台为例，就有select，poll，epoll三种，这三种的区别主要在于监听事件的底层方法不同，从而导致效率的差异

- select是早期Linux引入的io复用函数，底层采用轮询的方法判断每个文件描述符是否被激活。所谓轮询就是一遍遍的遍历，依次判断每一个文件描述符的状态，效率可想而知，慢
- poll是在select之后引入的，使用方法上稍微简单于select，但是仍然没有摆脱轮询带来的问题
- epoll作为轮询的终结者，底层没有采用轮询的方法，而是基于事件回调的。简单的说，就是在内核中当文件描述符被激活时都会调用一个回调函数，epoll根据回调函数直接定位文件描述符，极大提高了效率，同时也减轻了CPU的负担，不用一遍遍轮询



当然，除了效率问题，三者在使用上也是存在诸多差异

## select接口

```c
/* 
 * maxfds    : 最大的文件描述符 + 1
 * readfs    : 可读事件集
 * writefds  : 可写事件集
 * exceptfds : 其它(错误)事件集
 * tvptr     : 超时时间
 */
int select(int maxfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* tvptr);
```

其中fd_set结构保存的是需要监听的文件描述符，select将可读，可写，其它(错误)事件分开监听，返回被激活描述符的个数。但是仍需要一个一个遍历使用FD_ISSET判断是否被激活



## poll接口

```c
/* 
 * fdarray[] : 监听事件集
 * nfds      : 监听事件个数
 * timeout   : 超时时间
 */
int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);
```

在pollfd结构中保存需要监听的文件描述符，需要监听的事件，激活原因。使用起来比select简便的多



## epoll接口

```c
/* 
 * epollfd   : epoll文件描述符，用于监听所有的注册事件
 * events    : 保存所有激活事件
 * maxevents : events最大可容纳的激活事件个数
 * timeout   : 超时时间
 */
int epoll_wait(int epollfd, struct epoll_event* events, int maxevents, int timeout);
```

epoll_event结构保存了监听的文件描述符，监听的事件以及激活原因，与select和poll不同的是，epoll_wait直接将所有激活的事件保存在events中，这样就不需要一个个遍历判断哪个激活了



# Redis对io多路复用的封装

接下来以epoll为例，了解Redis内部是如何封装io多路复用的

为了将所有io复用统一，Redis为所有io复用统一了类型名aeApiState，对于epoll而言，类型成员就是调用epoll_wait所需要的参数

```c
//ae_epoll.c
typedef struct aeApiState {
    int epfd; //epollfd，文件描述符
    struct epoll_event *events; //保存激活的事件(epoll_event)
} aeApiState;
```

为什么保存两个就够了呢，epoll_wait明明需要4个参数。原因是在Redis初始化时，已经将保存激活事件的数组(events)的容量调至最大，所以maxevents只需要设置成最大即可，无需保存。对于超时时间，Redis的策略是在时间事件中找到最早超时的那个，计算还有多久到达超时时间，将这个时间差(相对时间)作为io复用的超时时间

这么设计的原因是如果Redis中没有时间事件，那么io复用函数可以一直阻塞在那里直到有事件被激活，如果有时间事件，为了不影响超时事件的回调，需要在事件超时时从io复用中返回，那么设置成超时时间是最合适的(这一点和libevent的策略相同)



接下来就是一些对epoll接口的封装了，包括创建epoll(epoll_create)，注册事件(epoll_ctl)，删除事件(epoll_ctl)，阻塞监听(epoll_wait)等

创建epoll就是简单的为aeApiState申请内存空间，然后将返回的指针保存在事件驱动循环中

```c
//ae_epoll.c
/* 创建epollfd，即调用::epoll_create */
static int aeApiCreate(aeEventLoop *eventLoop) {
    /* 申请内存 */
    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;
    /* events用于保存激活的事件，需要足够大的空间(不小于epoll_create时传入的参数) */
    /* eventLoop->setsize是初始化时设置的最大文件描述符个数 */
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
    if (!state->events) {
        zfree(state);
        return -1;
    }
    /* 创建epoll文件描述符 */
    state->epfd = epoll_create(1024); /* 1024 is just a hint for the kernel */
    if (state->epfd == -1) {
        zfree(state->events);
        zfree(state);
        return -1;
    }
    /* 保存io复用数据成员到事件驱动中 */
    eventLoop->apidata = state;
    return 0;
}
```

注册事件和删除事件就是对epoll_ctl的封装，根据操作不同选择不同的参数，以注册事件为例

```c
//ae_epoll.c
/* 
 * 将文件描述符和对应事件注册到io多路复用中
 * 即调用::epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event)
 */
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    /* 从事件驱动中获取io复用 */
    aeApiState *state = eventLoop->apidata;
    /* 用于传给epoll_ctl的参数 */
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    /* If the fd was already monitored for some event, we need a MOD
     * operation. Otherwise we need an ADD operation. */
    /* 判断是否是第一次注册，如果是则添加否则是修改 */
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;
    /* 合并以前的监听事件，因为不一定是首次添加 */
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    /* 根据监听事件的不同设置struct epoll_event中的events字段 */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    /* 保存监听的文件描述符 */
    ee.data.fd = fd;
    /* 调用接口 */
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}
```

阻塞监听是对epoll_wait的封装，在返回后将激活的事件保存在事件驱动中

```c
//ae_epoll.c
/* 阻塞监听，即调用::epoll_wait(epollfd, struct epoll_event*, int, struct timeval*); */
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;
    
    /* 时间单位是毫秒 */
    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    /* 有事件被激活 */
    if (retval > 0) {
        int j;

        numevents = retval;
        /* 保存所有激活的事件，将其文件描述符和激活原因保存在fired数组中 */
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            /* fired数组中只保存文件描述符和激活原因
             * 当需要获取激活事件时，根据文件描述符从eventLoop->events数组中查找 */
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    /* 返回激活事件的个数 */
    return numevents;
}
```



# 事件驱动循环流程

io复用的封装实现完成，那么Redis是何时调用io复用函数的呢，这就需要从server.c/main函数入手，可以猜测到当main函数初始化工作完成后，就需要进行事件驱动循环，而在循环中，会调用io复用函数进行监听

在初始化完成后，main函数调用了aeMain函数，传入的参数就是服务器的事件驱动

```c
//server.c
int main(int argc, char **argv) {
    /* 一系列的初始化工作 */
    ...
    aeMain(server.el);
    ...
}
```

在ae_epoll.c中可以找到aeMain函数，这个函数便是一直在循环，每次循环会调用aeProcessEvents函数

```c
//ae_epoll.c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    /* 一直循环监听 */
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

可以猜测，aeProcessEvents函数中一定调用io复用函数进行监听，当io复用返回后，执行每个激活事件的回调函数，这个函数比较长，但是还是蛮好理解的

```c
/* 每次事件循环都会调用一次该函数 */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

	if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;
	
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        /* 为io复用函数寻找超时时间(通常是最先超时的时间事件的时间(相对时间)) */
        /* redis中有时间事件 */
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        /* 根据最早超时的那个时间事件获取超时的相对时间 */
        if (shortest) {
            long now_sec, now_ms;

            /* 获取当前时间 */
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;

            /* 计算时间差(相对时间) */
            long long ms =
                (shortest->when_sec - now_sec)*1000 +
                shortest->when_ms - now_ms;

            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                tvp->tv_sec = 0;
                tvp->tv_usec = 0;
            }
        } else {
            /* 如果没有时间事件，那么io复用要么一直等，要么不等，取决于flags的设置 */
            /* 传入的struct timeval*是NULL表示一直等直到有事件被激活
             * 传入的timeval->tv_src = timeval->tv_usec = 0表示不等，直接返回 */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }

        /* 调用io复用函数，返回被激活事件的个数，所有被激活的事件保存在epollLoop->fired数组中 */
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            /* fired只保存的文件描述符和激活原因，实际的文件事件仍需要从events数组中取出 */
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

            /* 根据激活原因调用回调函数(先执行可读，再执行可写) */
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
    }

    /* 处理可能超时的时间事件 */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

至此一次事件驱动循环就执行完毕，里面的细节比较多，比如如何为io复用函数寻找超时时间，如果从激活事件调用回调函数，如果处理已超时事件等

> Redis对于时间事件是采用链表的形式记录的，这导致每次寻找最早超时的那个事件都需要遍历整个链表，容易造成性能瓶颈。而libevent是采用最小堆记录时间事件，寻找最早超时事件只需要O(1)的复杂度

# 如何选择合适的io多路复用函数

到目前位置还有一个问题没有解决，既然有那么多io复用函数，Redis怎么知道应该选择哪个呢，Redis的策略是选择当前平台存在的，效率最高的io复用函数

```c
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```



# 小结

其实任何一个基于网络请求的程序在这部分的内容都是相似的，无非就是Reactor模式的实现，不过毕竟Redis主要内容在数据库方面，网络这一块不会太过苛刻，如果只是想要学习服务器设计，可以参考libevent(C语言)，muduo(C++语言)