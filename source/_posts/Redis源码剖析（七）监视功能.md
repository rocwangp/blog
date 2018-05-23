---
title : Redis源码剖析（七）监视功能
date : 2018-01-09 15:26
categories : Redis
tags : Redis
---



Redis提供这样一个功能，客户端在开启事务之前，可以设置对一个或多个键的监视，在执行EXEC命令之前的这段时间，如果其他客户端对该客户端监视的键做了修改，那么Redis会取消该客户端事务的运行，也就是说如果执行EXEC，那么Redis什么也不会做。

由于别的客户端对当前客户端关心的键做了修改，Redis会将当前客户端的事务视为不安全，从而不再执行。Redis设计与实现一书中将监视功能称作乐观锁

<!--more-->

# 监视命令

监视功能由WATCH命令实现，可以设置对一个或多个键的监视

```c
127.0.0.1:6379> set time 14:33	//设置键值对
OK
127.0.0.1:6379> WATCH time	//设置对键time的监视
OK
127.0.0.1:6379> MULTI	//开启事务
OK
127.0.0.1:6379> set db redis
QUEUED
127.0.0.1:6379> get db
QUEUED
127.0.0.1:6379> get time
QUEUED
127.0.0.1:6379> 	//此时还处于事务状态，没有执行EXEC命令
```

此时开启另一个客户端对键time进行修改

> 注：修改的意思通常是改变键对应的值，使用各种SET命令

```c
127.0.0.1:6379> set time 14:34	//修改键time的值
OK
127.0.0.1:6379> 
```

现在回到之前的客户端，如果执行EXEC命令，会发现事务队列中的命令没有执行，而是返回了nil

```c
//执行事务之前
127.0.0.1:6379> set time 14:33
OK
127.0.0.1:6379> WATCH time
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set db redis
QUEUED
127.0.0.1:6379> get db
QUEUED
127.0.0.1:6379> get time
QUEUED
//执行事务之后
127.0.0.1:6379> EXEC		//执行事务，返回空
(nil)
127.0.0.1:6379> 
```

# 存储结构

要想知道一个客户端都监视了哪些键，就需要将其记录下来，Redis采用字典记录每个被监视的键和监视该键的所有客户端，不过这个字典不在redisServer结构中，而是在数据库redisDb结构中，该结构中保存了很多字典，其中就有数据键值对字典和过期时间字典，之前提到过的

```c
//server.h
typedef struct redisDb {
    dict *dict;            /* 保存键值对的字典 */ 
    dict *expires;         /* 保存键和其到期时间 */  
    dict *watched_keys;    /* 监视字典，保存每个被监视的键和所有监视该键的客户端 */ 
    ...
} redisDb;
```

监视字典中保存了所有客户端的监视信息，键是每个被监视的键，值是监视该键的客户端链表

和订阅模块相同，客户端同时也会记录自己监视了哪些键，不过客户端不需要使用字典，使用链表就够了。在client结构中，可以找到相关的定义

```c
//server.h
typedef struct client {
    list *watched_keys;     //监视链表，记录当前客户端监视的所有键
	...
} client;
```

此外，客户端的监视链表中并不是单单保存键，而是保存一个watchedKey结构，其中记录着监视的键和键所在的数据库

```c
//multi.c
/* 客户端监视链表中保存的结构，记录监视的键和键所在的数据库 */
typedef struct watchedKey {
    robj *key;
    redisDb *db;
} watchedKey;
```

# 监视功能的实现

## 添加监视的键

监视功能由watchCommand函数实现，函数中主要是将WATCH命令的参数依次添加到数据库的监视字典和客户端的监视链表中，由watchForKey函数完成

```c
//multi.c
/* 对键key进行监听，需要更新数据库的监视字典和客户端的监视链表 */
void watchForKey(client *c, robj *key) {
    list *clients = NULL;
    listIter li;
    listNode *ln;
    watchedKey *wk;

    /* 对客户端的监视链表进行遍历，判断键key是否已经被监视 */
    listRewind(c->watched_keys,&li);
    while((ln = listNext(&li))) {
        /* 取出链表的节点 */
        wk = listNodeValue(ln);
        /* 判断是否监视过键key */
        if (wk->db == c->db && equalStringObjects(key,wk->key))
            return; /* Key already watched */
    }
    /* 从数据库的监视字典中取出键key对应的客户端链表，如果不存在，则创建一个 */
    clients = dictFetchValue(c->db->watched_keys,key);
    if (!clients) {
        /* 不存在客户端链表(当前没有客户端对该键进行监听)，创建一个客户端链表作为键key对应的值 */
        clients = listCreate();
        dictAdd(c->db->watched_keys,key,clients);
        incrRefCount(key);
    }
    /* 将当前客户端追加到客户端链表中 */
    listAddNodeTail(clients,c);

    /* 增加键key到客户端的监视链表中 */
    wk = zmalloc(sizeof(*wk));
    wk->key = key;
    wk->db = c->db;
    incrRefCount(key);
    listAddNodeTail(c->watched_keys,wk);
}
```

## 修改被监视的键对事务的影响

当客户端开始监视功能后，其他客户端任何对监视键的修改都会破坏当前客户端的事务状态，导致Redis不再执行这次的事务。所以在对键进行修改的命令中一定有对监视键的处理，以SET命令为例，可以看到在setKey函数中执行了signalModifiedKey函数，目的是将所有监视该键的客户端的事务状态标记为已破坏(Redis不会执行已破坏的事务)

```c
//db.c
/* 添加或覆盖键值对 */
void setKey(redisDb *db, robj *key, robj *val) {
    if (lookupKeyWrite(db,key) == NULL) {
        dbAdd(db,key,val);
    } else {
        dbOverwrite(db,key,val);
    }
    incrRefCount(val);
    removeExpire(db,key);
    /* 因为对键key进行了修改，所以会导致监听该键的客户端事务被破坏
     * 调用该函数更改这些客户端的事务状态 */
    signalModifiedKey(db,key);
}
```

signalModifiedKey又调用touchWatchedKey函数，完成实际的修改任务。该函数将所有监视该键的客户端的事务标志设置为已破坏，当客户端输入EXEC命令时，Redis会先判断事务状态，如果已破坏，则不再执行事务队列中的命令

```c
//multi.c
/* 扫描服务器的监视字典，检查键key是否被某些客户端监视，如果有，将对应客户端标记为事务破坏状态 */
void touchWatchedKey(redisDb *db, robj *key) {
    list *clients;
    listIter li;
    listNode *ln;

    /* 如果数据库中没有键被监视，则返回 */
    if (dictSize(db->watched_keys) == 0) return;
    /* 尝试从监视字典中取出键key对应的值 */
    clients = dictFetchValue(db->watched_keys, key);
    /* 如果不存在，说明没有客户端监视该键，直接返回 */
    if (!clients) return;

    /* 设置迭代器方向从头到尾，开始遍历监视键key的所有客户端 */
    listRewind(clients,&li);
    while((ln = listNext(&li))) {
        /* 取出节点对应的值 */
        client *c = listNodeValue(ln);

        /* 设置客户端的事务状态，表示该客户端的事务已经被破坏，
         * 如果该客户端使用EXEC执行事务，则什么也不做直接返回 */
        c->flags |= CLIENT_DIRTY_CAS;
    }
}
```

在EXEC命令处理函数中，可以看到对于事务状态的判断

```c
/* 启动事务命令 */
void execCommand(client *c) {
    ...
    
    /* CLIENT_DIRTY_CAS标识代表客户端监视的键是否被修改过
     * 如果被修改过，说明事务已被破坏，那么执行事务就不再安全，直接返回 */
    if (c->flags & (CLIENT_DIRTY_CAS|CLIENT_DIRTY_EXEC)) {
        addReply(c, c->flags & CLIENT_DIRTY_EXEC ? shared.execaborterr :
                                                  shared.nullmultibulk);
        discardTransaction(c);
        goto handle_monitor;
    }

    /* 开始执行事务，监视任务就可以结束了，将该客户端的监视字典清空 */
    unwatchAllKeys(c); 
    ...
}
```

可以看到，在EXEC命令处理函数中有unwatchAllKeys这样一个函数调用，原因是监视功能只对单次事务有效，当事务结束后，当前客户端所有的监视也都会被清空。如果需要再进行监视，需要重新设置，当然，这就涉及到下一轮的事务



# 小结

监视功能和事务模块是结合在一起的，如果想要对事务进行保护，确保当其他客户端修改了某些必要的键时取消事务，就可以使用监视功能。另外，UNWATCH命令用于取消对一个或多个键的监视，不过该命令只能在事务之外执行，如果在事务状态中使用，则会被添加到事务队列中