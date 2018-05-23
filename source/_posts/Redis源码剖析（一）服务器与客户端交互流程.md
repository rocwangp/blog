---
title : Redis源码剖析（一）服务器与客户端交互流程
date : 2018-01-04
categories : Redis
tags : Redis
---



Redis底层还是基于网络请求的，对于单机数据库而言，网络请求仅仅是在一台机器上交互，即服务器客户端都在一台计算机上

当在终端输入redis-serve时，便启动了一个Redis服务器，随后开始初始化内部数据，对于Redis而言包括

- 读取配置文件初始化内部参数
- 创建默认数据库(默认为16个)
- 创建监听套接字并绑定回调函数(接收客户端连接请求)
- 执行事件驱动循环，开始响应客户端请求
- ...

<!--more-->

当在终端输入redis-cli时，便启动了一个客户端

到这里可以简单的猜测一下，Redis的命令交互流程大致为

- 启动一个客户端，请求连接到服务器
- 服务器接收客户端请求，建立连接成功，服务器开始监听客户端文件描述符并绑定回调函数
- 客户端输入命令，导致服务器端监听的文件描述符变为可读，服务器开始读取命令
- 服务器解析命令，并调用对应的命令处理函数
- 服务器将处理结果反馈给客户端
- 客户端文件描述符变为可读，读取反馈信息，输出在终端



上述是一个常见的C/S模型，Redis采用Reactor模式处理连接，Reactor模式就是常说的使用io多路复用函数监听客户端的方法。不过Redis是单线程下的Reactor，在常见的高并发服务器设计模型中可以使用Reactor+线程池的方法提高并发性(也叫one loop per thread，muduo网络库采用的设计模型)



下面就从源代码的角度体会服务器和客户端交互的流程(只截取关键部分)

# 服务器与客户端的交互流程

## 服务器监听客户端连接

当服务器启动时，首先执行的是server.c/main函数，如上所述，main函数进行了大量初始化工作，其中有一项是调用initServer函数

```c
//server.c
int main(int argc, char **argv) {
	...
	/* 初始化服务器，创建监听套接字 */
    initServer();
    ...
}
```

initServer函数中同样进行了大量初始化工作，其中的一部分是创建监听套接字

```c
//server.c
/* 服务器启动时调用，初始化服务器 */
void initServer(void) {
	...
    /* 创建监听套接字 */
    /* ipfd_count是监听套接字的数量 */
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
    ...
}
```

创建监听套接字由函数aeCreateFileEvent函数实现

```c
//ae.c
/* 
 * 创建文件事件(事件驱动) 
 * eventLoop : 服务器的事件驱动循环数组
 * fd        : 文件描述符
 * mask      : 需要监听的事件
 * proc      : 回调函数
 * clientData: 传给回调函数的参数
 */
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    /* 如果要创建的文件描述符大于服务器规定的大小，则报错 */
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
   
    /* 返回服务器事件驱动循环中的第fd个事件 */
    aeFileEvent *fe = &eventLoop->events[fd];

    /* 将文件描述符和其需要监听的事件添加到io多路复用函数中 */
    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    /* 将监听事件保存在事件驱动中 */
    fe->mask |= mask;
    /* 设置回调函数 */
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    /* 设置参数 */
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

函数中的aeEventLoop是事件驱动循环，保存所有正在监听的事件，当io复用返回时，会将所有以激活的事件也保存在aeEventLoop中以便于处理，和libevent中的事件驱动循环作用相同

aeFileEvent是对事件的封装，内部保存有监听的文件描述符，监听的事件以及回调函数，和libevent中的事件(event)作用相同

## 接收客户端连接请求

另外对于传给aeCreateFileEvent函数的回调函数，可以猜测它的作用主要就是调用accept函数接收客户端连接请求，建立与客户端关联的文件描述符，注册到io复用中。它的定义如下

```c
//networking.c
/* 服务器监听套接字的回调函数，用于接收客户端的连接请求 */
void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    int cport, cfd, max = MAX_ACCEPTS_PER_CALL;
    char cip[NET_IP_STR_LEN];

    while(max--) {
        /* 调用accept接收客户端连接请求，返回与客户端关联的文件描述符 */
        cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
        ...
        /* 根据文件描述符创建客户端实例(client对象)，用于与客户端交互 */
        acceptCommonHandler(cfd,0,cip);
    }
}

```

每当接收到一个客户端请求时，服务器都会根据客户端文件描述符创建一个客户端实例(client类型)，client是服务器与客户端交互的桥梁，客户端输入的所有命令都是读取到client中的缓冲区再进行处理的

## 创建客户端实例

创建客户端实例的函数定义如下

```c
//networking.c
/* 
 * 根据客户端文件描述符创建客户端实例
 * fd : 接收客户端连接请求时返回的文件描述符
 * ip : 客户端地址<ip, port>
 */
static void acceptCommonHandler(int fd, int flags, char *ip) {
    client *c;
    /* 以客户端文件描述符创建客户端实例 */
    if ((c = createClient(fd)) == NULL) {
        serverLog(LL_WARNING,
            "Error registering fd event for the new client: %s (fd=%d)",
            strerror(errno),fd);
        close(fd); /* May be already closed, just ignore errors */
        return;
    }
    ...
}
```

该函数调用createClient，执行真正创建客户端实例的操作

```c
//networking.c
/* 根据文件描述符创建与客户端的连接 */
client *createClient(int fd) {
    /* 申请client大小的内存空间 */
    client *c = zmalloc(sizeof(client));
    if (fd != -1) {
        /* 设置成非阻塞io */
        anetNonBlock(NULL,fd);
        anetEnableTcpNoDelay(NULL,fd);
        /* keep-alive选项 */
        if (server.tcpkeepalive)
            anetKeepAlive(NULL,fd,server.tcpkeepalive);
        /* 为这次连接创建事件，监听可读事件, 回调函数为readQueryFromClient */
        if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)
        {
            close(fd);
            zfree(c);
            return NULL;
        }
    }
	...
}
```

创建与文件描述符关联的事件和上面相同，只是这里的回调函数变为readQueryFromClient，因为这个连接不是为了接收客户端连接请求，而是用于接收客户端的输入命令。

## 处理客户端输入的命令

当客户端输入命令后，会执行相应的回调函数readQueryFromClient，该函数主要调用read函数从客户端文件描述符中读取输入的命令

```c
//networking.c
/* 当客户端可读时的回调函数 */
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    client *c = (client*) privdata;
    int nread, readlen;

    readlen = PROTO_IOBUF_LEN;
    /* 为缓冲区申请空间，用于保存客户端命令 */
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    /* 从客户端读取命令，保存在c->querybuf中 */
    nread = read(fd, c->querybuf+qblen, readlen);
    ...
    /* 处理客户端命令 */
    processInputBuffer(c);
```

processInputBuffer函数根据客户端选项执行不同的操作，最终调用processCommand函数处理命令，这个函数会先解析客户端的命令关键字，判断这个关键字是否合法。如果合法，再判断参数个数是否合法

```c
//server.c
/* 解析客户端命令，先判断命令关键字是否合法，再判断参数个数是否合法 */
int processCommand(client *c) {
	...
    /* 从命令字典中查找该命令名字，判断是否存在该命令 */
    /* 将命令保存在cmd中，其中包括命令对应的处理函数 */
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    ...
    if(...)
    {
      	...
    }
    else
    {
    	//调用命令处理函数 
        call(c,CMD_CALL_FULL);
        ...
    }
}
```

可以看到这个函数主要是解析命令关键字，从底层字典中查找是否有这个关键字，如果有，会连同命令对应的处理函数一起返回赋值给cmd变量中，cmd变量是struct redisCommand *类型，稍后可以看到用处

call函数的定义如下

```c
//server.c
/* 调用命令处理函数 */
void call(client *c, int flags) {
    ...
    /* 调用命令回调函数 */
    c->cmd->proc(c);
    ...
}
```

## 命令结构

Redis内部已经将每个命令以及其对应的处理函数包装好，这个结构就是struct redisCommand，其中两个比较重要的成员变量为

```c
//server.h
struct redisCommand {
    char *name; //命令关键字
    redisCommandProc *proc;//处理函数
    ...
};
```

到这里可以猜到，在processCommand函数中调用lookupCommand函数时，会查找是否有相应命令关键字的结构，如果有，则返回到cmd变量中。现在可以看一下Redis内部是如何包装每一个命令的了

```c
struct redisCommand redisCommandTable[] = {
    {"get",getCommand,2,"rF",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,NULL,1,1,1,0,0},
    {"setnx",setnxCommand,3,"wmF",0,NULL,1,1,1,0,0},
    {"setex",setexCommand,4,"wm",0,NULL,1,1,1,0,0},
    {"psetex",psetexCommand,4,"wm",0,NULL,1,1,1,0,0},
	...
};
```

原来Redis已经为每个命令设计好了struct redisCommand结构，但是查找是否有命令关键字却不是直接从这个“超大”的数组中一个个找，那样太慢了。Redis内部是将每个命令关键字和它对应的struct redisCommand结构记录在一个字典中，由于字典(底层由哈希表实现)的查找效率是O(1)，所以不会造成性能瓶颈

获取到相应的命令结构后，同样也获取了命令处理函数，对于get命令而言是getCommand，对于set命令而言是setCommand，大概就是命令关键字后加Command是对应的命令处理函数

所以上述c->cmd->proc(c)函数调用时便直接调用了相应的处理函数，处理完成后，将反馈发送给客户端，完成一次交互



# 小结

服务器与客户端的交互实际上还是基于网络请求的，服务器监听客户端请求，客户端请求连接，当连接建立成功后服务器会开始监听客户端的文件描述符(套接字)，一旦客户端输入命令，服务器便读取文件描述符获得客户端的输入，然后解析，执行处理函数，将结构反馈给客户端，客户端将结构显示在终端，完成一次交互。

不过上述列举的仅仅是冰山一角，源代码中有诸多细节值得品味，包括对io复用的封装，tcp连接的建立与维护，底层数据结构的实现等，都需要好好理解