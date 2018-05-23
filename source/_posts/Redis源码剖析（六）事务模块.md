---
title : Redis源码剖析（六）事务模块
date : 2018-01-06 22:23
categories : Redis
tags : Redis
---



Redis允许客户端开启事务模式，在事务模式中，客户端输入的命令不会立即执行而是被保存在事务队列中，只有当客户端输入事务运行命令时，Redis才会将事务队列中的所有命令按照FIFO的顺序一个个执行

一个事务从开始到结束通常会经历三个阶段

- 事务开始
- 命令入队
- 事务执行

<!--more-->



# 事务命令

客户端可以使用MULTI命令开启事务，随后服务器会根据这个客户端输入的不同命令执行不同的操作

- 如果客户端发送的命令为EXEC，DISCARD，WATCH，MULTI四个命令中的一个，那么服务器立刻执行这个命令，同时是否关闭事务取决于每个命令的功能
- 如果客户端发送的命令为上述四个命令之外的其他命令，那么服务器不会立刻执行输入的命令，而是将命令存放在事务队列中

```c
127.0.0.1:6379> MULTI	//开启事务
OK
127.0.0.1:6379> set db redis
QUEUED	//表示命令被添加到事务队列中
127.0.0.1:6379> get db
QUEUED
127.0.0.1:6379> set name roc
QUEUED
127.0.0.1:6379> get name
QUEUED
127.0.0.1:6379> EXEC	//执行事务队列中的命令
1) OK			//set db redis的执行结果
2) "redis"		//get db的执行结果
3) OK			//set name roc的执行结果
4) "roc"		//get name的执行结果
127.0.0.1:6379> 
```

# 存储结构

要想当输入EXEC时一次性执行之前输入的所有命令，就需要将之前的命令保存起来，Redis采用数组保存所有命令信息，在client的定义中可以找到

```c
//server.h
typedef struct client {
    ...
    multiState mstate;   /* 事务属性，保存事务队列以及事务状态 */
    ...
} client;
```

multiState是事务属性结构，它保存着事务队列以及事务队列中命令个数，定义如下

```c
//server.h
/* 事务属性 */
typedef struct multiState {
    multiCmd *commands;  /* 事务队列，保存多条命令 */   
    int count;          /* 事务队列中命令个数 */ 
    ...
} multiState;
```

commands是一个事务队列，实际上是一个multiCmd类型的数组，multiCmd结构保存一条命令的信息

```c
//server.h
/* 保存一条命令的信息 */
typedef struct multiCmd {
    robj **argv; /* 命令关键字和参数 */
    int argc; /* 命令参数个数 */
    struct redisCommand *cmd; /* 命令结构，主要包含命令处理函数 */
} multiCmd;
```

可以看到，multiCmd中保存的实际上就是执行一条命令需要的三个信息，分别是

- 命令关键字和参数
- 参数个数
- 命令处理函数

在客户端client的定义中也可以找到这三个变量的定义

```c
//server.h
typedef struct client {
    ...
    int argc;               
    robj **argv;           
    struct redisCommand *cmd;  
    ...
} client;
```

所以大体可以猜测，当要执行事务队列中的命令时，只需要遍历事务队列，依次将这三个变量赋值给client中的对应变量，然后调用对应命令处理函数即可。后面会看到，事实也正是如此



# 事务实现

## 开启事务

在Redis内部，事务的开启十分简单，仅仅是将客户端的事务标志位打开，表示进行事务状态，随后的大多数操作都会先判断该标志位以确定是将命令添加到事务队列中，还是执行命令

```c
//multi.c
/* 开启事务 */
void multiCommand(client *c) {
    /* 若已经开启，则报错 */
    if (c->flags & CLIENT_MULTI) {
        addReplyError(c,"MULTI calls can not be nested");
        return;
    }
    /* 设置CLIENT_MULTI标志代表客户端已经开启事务 */
    c->flags |= CLIENT_MULTI;
    addReply(c,shared.ok);
}
```

## 添加命令到事务队列

在第一篇[服务器与客户端交互流程](https://rocwangp.github.io/2018/01/04/Redis%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90%EF%BC%88%E4%B8%80%EF%BC%89%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8E%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BA%A4%E4%BA%92%E6%B5%81%E7%A8%8B/#more)中得知，当客户端输入命令后，会存在一个解析命令的操作，将命令参数，参数个数以及命令处理函数找到，然后执行call函数，在这个函数中调用命令处理函数执行命令。但是一旦开启事务功能，Redis就不能再执行命令了，如上所述，应该将命令添加到事务队列中。

在processCommand函数中，可以看到对于这两种情况的判断

```c
/* 处理客户端输入的命令 */
int processCommand(client *c) {
    ...
    
    /*　从命令字典中查找该命令名字，返回redisCommand结构，其中包含命令处理函数 */
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);

    ...
    
    /* 如果客户端开启事务，则不执行命令而是将命令添加到事务队列中 */
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        queueMultiCommand(c);
        /* 回复客户端当前命令已经添加到事务队列中 */
        addReply(c,shared.queued);
    } else {
    	/* 没有开启事务，执行执行命令 */
        call(c,CMD_CALL_FULL);
        ...
    }
    return C_OK;
}
```

将命令添加到事务队列中由queueMultiCommand函数完成，函数首先创建一个multiCmd对象，这个结构在上面提到过，保存着执行一条命令需要的三个元素，分别是命令参数，参数个数以及命令处理函数。而multiState结构是保存事务队列的结构(实际是数组)，在这里需要将新的命令添加到这个数组中

```c
/* 将当前命令添加到客户端的事务队列中 */
void queueMultiCommand(client *c) {
    multiCmd *mc;
    int j;

    /* mstate是multiState类型，保存事务中所有命令的信息
     * 每增加一条命令到事务队列中，都需要为原事务队列重新申请n+1大小的空间
     * 多的那一个用来存储当前命令*/
    c->mstate.commands = zrealloc(c->mstate.commands,
            sizeof(multiCmd)*(c->mstate.count+1));
    mc = c->mstate.commands+c->mstate.count;
    /* 保存执行一条命令所需的三个元素 */
    mc->cmd = c->cmd;
    mc->argc = c->argc;
    mc->argv = zmalloc(sizeof(robj*)*c->argc);
    /* 将参数复制到multiCmd结构中 */
    memcpy(mc->argv,c->argv,sizeof(robj*)*c->argc);
    /* 因为参数是robj*类型，所以引用计数加一 */
    for (j = 0; j < c->argc; j++)
        incrRefCount(mc->argv[j]);
    /* 事务队列中命令个数加一 */
    c->mstate.count++;
}
```

## 执行命令

当客户端输入EXEC命令后，Redis会从客户端的事务队列中取出命令，按照保存的先后顺序一个个执行，由于事务队列中有命令的所有信息，所以可以执行调用处理函数。这部分操作由execCommand函数执行，因为执行一条命令是从客户端对象client中取出命令参数，参数个数以及命令处理函数，所以这里就直接将队列中的命令信息复制给客户端对象的对应变量，然后和执行正常命令一样调用call函数

```c
/* 启动事务命令 */
void execCommand(client *c) {
    int j;
    robj **orig_argv;
    int orig_argc;
    struct redisCommand *orig_cmd;
    int must_propagate = 0; /* Need to propagate MULTI/EXEC to AOF / slaves? */

    /* CLIENT_MULTI标识代表当前客户端是否开启事务，如果没有开启，执行EXEC指令是没有意义的 */
    if (!(c->flags & CLIENT_MULTI)) {
        addReplyError(c,"EXEC without MULTI");
        return;
    }

    /* CLIENT_DIRTY_CAS标识代表客户端监视的键是否被修改过
     * 如果被修改过，那么执行事务就不再安全，直接返回 */
    if (c->flags & (CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC)) {
        addReply(c, c->flags & CLIENT_DIRTY_EXEC ? shared.execaborterr :
                                                  shared.nullmultibulk);
        discardTransaction(c);
        goto handle_monitor;
    }

    /* Exec all the queued commands */
    /* 开始执行事务，监视任务就可以结束了，将该客户端的监视字典清空 */
    unwatchAllKeys(c); /* Unwatch ASAP otherwise we'll waste CPU cycles */
    /* 临时保存客户端当前参数信息 */
    orig_argv = c->argv;
    orig_argc = c->argc;
    orig_cmd = c->cmd;
    addReplyMultiBulkLen(c,c->mstate.count);
    /* 开始执行客户端数据队列(数组)中的命令 */
    for (j = 0; j < c->mstate.count; j++) {
        /* 将每个命令作为客户端当前的参数信息(这就是为什么需要临时保存以前参数的原因) */
        c->argc = c->mstate.commands[j].argc;
        c->argv = c->mstate.commands[j].argv;
        c->cmd = c->mstate.commands[j].cmd;

        ...

        /* 开始执行命令 */
        call(c,CMD_CALL_FULL);

        /* Commands may alter argc/argv, restore mstate. */
        /* call执行的命令可能会改变事务队列中的当前命令，这里是确保其不被改变 */
        c->mstate.commands[j].argc = c->argc;
        c->mstate.commands[j].argv = c->argv;
        c->mstate.commands[j].cmd = c->cmd;
    }
    /* 还原客户端以前的参数信息 */
    c->argv = orig_argv;
    c->argc = orig_argc;
    c->cmd = orig_cmd;
    ...
}
```

函数有点长，不过还算好理解，先判断客户端是否开启事务(利用CLIENT_MULTI标记)，然后清空客户端的监视链表(后面会提到)，最后遍历事务队列，依次执行每个命令

需要注意的是，Redis在执行事务队列中的命令时保证即使某条命令是错误的，也不会影响到其他命令的执行，即如果中间某条命令执行出错，那么后面的命令仍然会继续执行。另外，Redis不支持事务回滚功能，即不支持将已执行的命令撤销

# 小结

Redis的事务功能还是比较简单的，另外需要提的是，由于Redis是运行在单线程下的，而且对于客户端的监听是基于io多路复用函数的，所以对于客户端的响应是串行的，不会出现当执行某个客户端事务队列中的命令时切换到另一个客户端的情况。这保证了事务执行具有原子性，另外Redis设计与实现书中还讲到Redis的事务具有一致性，隔离性和耐久性，有兴趣的话可以翻阅书籍查看