---
title : Redis源码剖析（五）订阅与发布
date : 2018-01-05 16:33
categories : Redis
tags : Redis
---



Redis提供了订阅和发布的功能，允许客户端订阅一个或多个频道，当其他客户端向某个频道发送消息时，服务器会将消息转发给所有订阅该频道的客户端

这一点有点像群聊的功能，一个客户端将消息发往群中(向某个频道发送消息)，所有在群中的客户端(订阅该频道的客户端)都会收到这个消息。事实也正是如此，接下来将会看到，服务器采用字典保存每个频道(键)和订阅该频道的所有客户端(值)，每当其他客户端向某个频道发送消息时，服务器便从字典中获取所有订阅该频道的客户端，依次将消息发送。每个频道，可以看成是每个群，一个频道的所有订阅客户端，可以看成该群的所有群成员，唯一不同的是，向频道发送消息的那个客户端并不需要订阅同样的频道，也就是该客户端并不需要也在群中

稍后会看到，除了订阅特定频道，Redis也允许客户端进行模式订阅，即一次订阅所有匹配的频道

<!--more-->



Redis的订阅与发布功能由PUBLISH, SUBSCRIBE, PSUBSCRIBE等命令组成

# 普通订阅

使用SUBSCRIBE [频道名]即可订阅特定频道，频道名可以自定义，也可以同时订阅多个频道，只需要后面添加多个频道名即可

```c
127.0.0.1:6379> SUBSCRIBE "news.redis"	//订阅"news.redis"频道
Reading messages... (press Ctrl-C to quit)
1) "subscribe"	//命令关键字
2) "news.redis"	//频道名
3) (integer) 1	//订阅该频道的客户端数量
```

使用PUBLISH [频道名] \[消息]即可向特定频道发送消息

```c
127.0.0.1:6379> PUBLISH "news.redis" "send a message"	//向"news.redis"频道发送消息
(integer) 1	//返回发送给了多少个客户端
127.0.0.1:6379> 
```

此时，如果再查看订阅news.redis频道的那个客户端，会发现终端上打印出"send a message"信息

```c
//PUBLISH之前
127.0.0.1:6379> SUBSCRIBE "news.redis"
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "news.redis"
3) (integer) 1

//PUBLISH之后
1) "message"	//消息类型
2) "news.redis"	//频道名
3) "send a message"	//信息
```

不过如果一个客户端处于订阅状态，它好像就不能执行其他操作了

## 存储结构

实现一个订阅与发布功能十分简单，开篇也提到了，只需要将每个频道以及它的订阅者记录在字典中，如果客户端向某个频道发送消息，则在字典中查找该频道的所有订阅者，依次将消息发送过去即可。

在深入源代码之前，先看两个结构的定义，一个是客户端，一个是服务器，它们都定义在server.h头文件中

```c
//server.h
typedef struct client {
	...
    dict *pubsub_channels; 
    ...
} client;
```

```c
//server.h
struct redisServer {
	...
    dict *pubsub_channels;  
    ...
};
```

这两个结构都太长了，不过目前用得到的其实就一个pubsub_channels变量，根据类型得知该变量是一个字典(以下简称为订阅字典)，两个变量的作用分别是

- 客户端的订阅字典记录着当前客户端订阅的所有频道，键是频道名，值为空
- 服务器的订阅字典记录着所有频道以及每个频道的订阅者，键是频道名，值是客户端链表

到这里其实可以简单猜测订阅功能是如何实现的，当某个客户端使用SUBSCRIBE命令订阅一个或多个频道时，Redis会将<频道名，客户端>这个键值对添加到服务器的订阅字典中，同时也会将频道名添加到客户端自己的订阅字典中

而当客户端使用PUBLISH命令向某个频道发送消息时，Redis会在订阅字典中获取该频道的所有订阅者(客户端)，依次将消息发送给客户端。如果该频道不存在或没有订阅者，则不执行任何操作

## 订阅功能

订阅功能由subscribeCommand函数完成，函数主要任务是遍历每一参数(频道名)，调用pubsubSubscribeChannel函数将频道名和客户端添加到订阅字典中

```c
//pubsub.c
/* 订阅命令 */
void subscribeCommand(client *c) {
    int j;

    /* 将客户端和它订阅的频道进行关联，添加到订阅字典中
     * 键是频道名，值是客户端 
     */
    for (j = 1; j < c->argc; j++)
        pubsubSubscribeChannel(c,c->argv[j]);
    /* 标记当前客户端订阅过某些频道 */
    c->flags |= CLIENT_PUBSUB;
}
```

pubsubSubscribeChannel函数完成实际的添加操作

```c
//pubsub.c
/* 
 * 将客户端和它订阅的频道进行关联，添加到客户端和服务器两个订阅字典中 
 * 
 * 注：服务器和客户端都有订阅字典，分别是
 * c->pubsub_channels
 * server.pubsub_channels
 */
int pubsubSubscribeChannel(client *c, robj *channel) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    /* 判断当前客户端是否已经订阅了该频道，如果是则不进行处理，否则添加到客户端的订阅字典中 */
    /* 注意这里添加的是客户端的订阅字典，该字典记录当前客户端订阅的所有频道 */
    if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) {
        retval = 1;
        /* 所有的robj对象都是基于引用计数的，因为已将其添加到字典中，所有引用计数加一 */
        incrRefCount(channel);
        /* 从服务器的订阅字典中寻找该频道对应的键节点链表(记录所有订阅该频道的客户端链表) */
        de = dictFind(server.pubsub_channels,channel);
        if (de == NULL) {
            /* 服务器订阅字典中没有关于该频道的记录，创建该频道对应的客户端链表 */
            clients = listCreate();
            /* 将<频道，客户端链表>添加到服务器的订阅字典中 */
            dictAdd(server.pubsub_channels,channel,clients);
            /* 频道的引用计数加一 ，因为在字典中也有一份*/
            incrRefCount(channel);
        } else {
            /* 服务器订阅字典中有关于该频道的记录，直接将客户端链表返回 */ 
            clients = dictGetVal(de);
        }
        /* 将当前客户端连接到链表上 */
        listAddNodeTail(clients,c);
    }
    /* 通知客户端订阅成功 */ 
    addReply(c,shared.mbulkhdr[3]);
    addReply(c,shared.subscribebulk);
    addReplyBulk(c,channel);
    addReplyLongLong(c,clientSubscriptionsCount(c));
    return retval;
}
```

至此订阅操作完成，可以发现订阅仅仅是将频道名和客户端这个键值对添加到订阅字典中，并不执行其他操作。

## 退订功能

有订阅就有退订，退订命令是UNSUBSCRIBE，有unsubscribeCommand函数执行。不过既然订阅功能是阻塞的，怎么执行退订啊...

退订分两种，一种是退订当前客户端订阅的所有频道，此时退订命令不带参数。另一种则带参数，仅退订参数指出的频道

```c
//pubsub.c
/* 退订命令 */
void unsubscribeCommand(client *c) {
    if (c->argc == 1) {
        /* 退订当前客户端订阅的所有频道 */
        pubsubUnsubscribeAllChannels(c,1);
    } else {
        int j;

        /* 退订参数指出的频道 */
        for (j = 1; j < c->argc; j++)
            pubsubUnsubscribeChannel(c,c->argv[j],1);
    }
    /* 客户端订阅的频道数为0时，改变标志 */
    if (clientSubscriptionsCount(c) == 0) c->flags &= ~CLIENT_PUBSUB;
}
```

退订所有频道是遍历当前客户端的订阅字典，对订阅的每个频道调用pubsubUnsubscribeChannel函数，实际上和指定参数效果相同，所以就直接看退订参数指定频道的函数好了

```c
//pubsub.c
/* 
 * 退订
 * c : 客户端
 * channel : 要退订的频道
 * notify : 退订后是否通知客户端
 */
int pubsubUnsubscribeChannel(client *c, robj *channel, int notify) {
    dictEntry *de;
    list *clients;
    listNode *ln;
    int retval = 0;

    incrRefCount(channel); /* channel may be just a pointer to the same object
                            we have in the hash tables. Protect it... */
    /* 从客户端订阅字典中删除关于该频道的订阅信息 */
    if (dictDelete(c->pubsub_channels,channel) == DICT_OK) {
        /* 删除成功，表示这个客户端订阅过channel */
        retval = 1;
        
        /* 从服务器订阅字典中查找关于该频道的所有订阅信息，返回键节点 */
        de = dictFind(server.pubsub_channels,channel);
        serverAssertWithInfo(c,NULL,de != NULL);
        /* 从键节点中获取客户端链表 */
        clients = dictGetVal(de);
        /* 从客户端链表中搜索当前退订的客户端 */
        ln = listSearchKey(clients,c);
        serverAssertWithInfo(c,NULL,ln != NULL);
        /* 将链表节点ln从链表clients中删除 */
        listDelNode(clients,ln);
        /* 如果该频道只有该客户端订阅过，那么删除后客户端链表为空，从服务器订阅字典中删除该频道的信息 */
        if (listLength(clients) == 0) {
            dictDelete(server.pubsub_channels,channel);
        }
    }
    /* 如果要求通知，则通知客户端 */
    if (notify) {
        addReply(c,shared.mbulkhdr[3]);
        addReply(c,shared.unsubscribebulk);
        addReplyBulk(c,channel);
        addReplyLongLong(c,dictSize(c->pubsub_channels)+
                       listLength(c->pubsub_patterns));

    }
    decrRefCount(channel); /* it is finally safe to release it */
    return retval;
}
```

退订函数虽然长了点，但是还是蛮好理解的，仅仅是将客户端和频道的关联信息从订阅字典中删除



# 普通订阅的信息发布

Redis的发布功能由PUBLISH命令实现，底层由pubsubPublishMessage函数实现，该函数向订阅特定频道的所有客户端发送消息。订阅分两种，一个是普通订阅(如上)，另一个是模式订阅，所以函数中也分为向普通订阅的客户端发送消息和向模式订阅的客户端发送消息。因为还没有接触模式订阅，所以先看普通订阅的发布好了

普通订阅的发送消息仅仅是在服务器的订阅字典中寻找特定频道的所有订阅者，依次将消息发送就完成了，比较简单

```c
//pubsub.c
/* 发送通知信息 */
/* 
 * channel : 通知信息
 * message : 事件名称
 */
int pubsubPublishMessage(robj *channel, robj *message) {
    int receivers = 0;
    dictEntry *de;
    listNode *ln;
    listIter li;

    /* 服务器的订阅字典保存着所有频道和它的所有订阅者 */
    /* 从该字典中查找频道channel的订阅者，返回键节点 */
    de = dictFind(server.pubsub_channels,channel);
    if (de) {
        /* 从键节点中获取订阅该频道的客户端链表 */
        list *list = dictGetVal(de); 
        listNode *ln;
        listIter li;

        /* 将迭代器方向设置为从头到尾 */
        listRewind(list,&li);
        /* 遍历客户端链表的所有客户端，发送通知信息 */
        while ((ln = listNext(&li)) != NULL) {
            client *c = ln->value;

            addReply(c,shared.mbulkhdr[3]);
            addReply(c,shared.messagebulk);
            addReplyBulk(c,channel);
            addReplyBulk(c,message);
            receivers++;
        }
    }
    ...
    return receivers;
}
```

# 模式订阅



Redis允许客户端使用正则表达式订阅一组频道，命令格式为PSUBSCRIBE [频道名]

```c
127.0.0.1:6379> PSUBSCRIBE "news.redi[xy]"	//订阅"news.redix"和"news.rediy"两个频道
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"	//命令关键字
2) "news.redi[xy]"	//频道名
3) (integer) 1	//订阅该频道的客户端数量
```

此时，如果打开另一个客户端，不管是向news.redix频道发送还是向news.rediy频道发送消息，上面这个客户端都会接收到消息

```c
127.0.0.1:6379> PUBLISH "news.redix" "send to news.redix"	//向"news.redix"频道发送消息
(integer) 1
127.0.0.1:6379> PUBLISH "news.rediy" "send to news.rediy"	//向"news.rediy"频道发送消息
(integer) 1
127.0.0.1:6379> 
```

```c
127.0.0.1:6379> PSUBSCRIBE "news.redi[xy]"	//订阅"news.redix"和"news.rediy"两个频道
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "news.redi[xy]"
3) (integer) 1
1) "pmessage"		//从频道news.redix接收到消息
2) "news.redi[xy]"	
3) "news.redix"		//频道名
4) "send to news.redix"	//消息内容
1) "pmessage"		//从频道news.rediy接收到消息
2) "news.redi[xy]"
3) "news.rediy"		//频道名
4) "send to news.rediy" //消息内容
```

> 以下将用正则表达式表示的频道称为频道组，如"news.redi[xy]"就是一个频道组

## 存储结构

由于模式订阅的频道名代表一组频道，所以不能用字典存储，因为字典的键是已知的，当然可以将用正则表达式代表的频道的所有可能都计算处然后添加到字典中，不过Redis不会这么做，论谁谁都不会，因为结果集太大了。

所以字典在这里算是没有用武之地了，Redis采用链表将每个客户端和它订阅的频道组记录起来，每当向特定频道发布消息时，Redis就会遍历这个链表判断每个客户端的频道组是否可以和当前频道匹配，如果匹配则向该客户端发送消息。当然，每个客户端也有这么个链表记录自己订阅的频道组，在它们的定义中可以清楚的看到

```c
//server.h
typedef struct client {
    dict *pubsub_channels; 	//订阅字典
    list *pubsub_patterns;  //模式订阅链表
} client;
```

```C
//server.h
struct redisServer {
    dict *pubsub_channels; 	//订阅字典
    list *pubsub_patterns;  //模式订阅链表
};
```

与订阅字典相同，模式订阅链表在客户端和服务器的作用也不相同

- 客户端的模式订阅链表保存当前客户端订阅的所有频道组
- 服务器的模式订阅链表保存所有客户端订阅的所有频道组(链表中可能有多个节点指向的客户端相同，但是频道组不同)

客户端链表的节点保存的是频道组

服务器链表的节点保存的结构是pubsubPattern类型，该结构记录着客户端和一个频道组

```c
//server.h
typedef struct pubsubPattern {
    client *client; //客户端
    robj *pattern;  //频道组
} pubsubPattern;
```

有了订阅模块的基础，到这里可以猜测模式订阅也仅仅是将客户端和其模式订阅的频道组组成pubsubPattern添加到服务器的模式订阅链表中，将频道组添加到客户端的模式订阅链表中，并不做其他处理

## 模式订阅功能

事实也正是如此，模式订阅功能由pubsubSubscribePattern函数实现

```c
//pubsub.c
/* 模式订阅 */
int pubsubSubscribePattern(client *c, robj *pattern) {
    int retval = 0;

    /* 从客户端的模式订阅链表中查找要订阅的模式，如果不存在，才进行添加 */
    if (listSearchKey(c->pubsub_patterns,pattern) == NULL) {
        retval = 1;
        /* pubsubPattern结构记录着客户端c和频道组pattern */
        pubsubPattern *pat;
        /* 将频道组添加到客户端模式订阅链表尾部 */
        listAddNodeTail(c->pubsub_patterns,pattern);
        incrRefCount(pattern);
        /* 申请空间，组装pubsubPattern结构 */
        pat = zmalloc(sizeof(*pat));
        pat->pattern = getDecodedObject(pattern);
        pat->client = c;
        /* 将订阅节点添加到服务器的订阅链表中 */
        listAddNodeTail(server.pubsub_patterns,pat);
        /* 因为客户端的订阅链表只需要记录自己订阅频道组即可，所以无需存储pubsubPattern结构
         * 而服务器需要记录每个客户端和其频道组，二者都需要记录，所以存储pubsubPattern结构 */
    }
    /* Notify the client */
    addReply(c,shared.mbulkhdr[3]);
    addReply(c,shared.psubscribebulk);
    addReplyBulk(c,pattern);
    addReplyLongLong(c,clientSubscriptionsCount(c));
    return retval;
}
```

## 模式退订功能

退订和订阅是相反的，对于模式订阅的退订也是如此，仅仅是将频道组从模式订阅链表中删除，需要注意的是要退订就退订整个频道组，Redis不支持将特定频道从频道组中去除

```c
//pubsub.c
/* 退订模式 */
int pubsubUnsubscribePattern(client *c, robj *pattern, int notify) {
    listNode *ln;
    pubsubPattern pat;
    int retval = 0;

    incrRefCount(pattern); /* Protect the object. May be the same we remove */
    /* 从客户端自己的模式订阅链表中查找相应模式 */
    if ((ln = listSearchKey(c->pubsub_patterns,pattern)) != NULL) {
        retval = 1;
        /* 如果找到，则删除链表节点 */
        listDelNode(c->pubsub_patterns,ln);
        pat.client = c;
        pat.pattern = pattern;
        /* 从服务器的模式订阅链表中查找，然后删除 */
        ln = listSearchKey(server.pubsub_patterns,&pat);
        listDelNode(server.pubsub_patterns,ln);
    }
    /* Notify the client */
    if (notify) {
        addReply(c,shared.mbulkhdr[3]);
        addReply(c,shared.punsubscribebulk);
        addReplyBulk(c,pattern);
        addReplyLongLong(c,dictSize(c->pubsub_channels)+
                       listLength(c->pubsub_patterns));
    }
    decrRefCount(pattern);
    return retval;
}
```

# 模式订阅的信息发布

最后一个就是关于向模式订阅发布消息的实现了，在上面订阅模块处，仅仅看到了pubsubPublishMessage函数向订阅特定频道的客户端发送消息。而实际上，它还有一部分是向模式订阅的客户端发送消息，方法是遍历模式订阅链表，对于每一个节点判断其频道组是否和当前频道匹配，如果匹配，则向客户端发送消息

```c
//pubsub.c
/* 发送通知信息 */
/* 
 * channel : 通知信息
 * message : 事件名称
 */
int pubsubPublishMessage(robj *channel, robj *message) {
    int receivers = 0;
    dictEntry *de;
    listNode *ln;
    listIter li;
	
	... //这里省略普通订阅的发布功能
	
    /* 查找模式订阅的客户端 */
    /* 这里就体现了为什么订阅频道和客户端是用字典存储，而模式订阅则用链表存储 
     * 因为订阅可以直接使用哈希表定位，而模式订阅类似正则匹配，需要判断当前的频道是否
     * 是匹配的模式订阅，然后发送给订阅者，哈希表在这里是没有作用的 */
    if (listLength(server.pubsub_patterns)) {
        /* 迭代器方向设置为从头到尾 */
        listRewind(server.pubsub_patterns,&li);
        channel = getDecodedObject(channel);
        /* 遍历服务器的模式订阅链表 */
        while ((ln = listNext(&li)) != NULL) {
            /* 获取每个节点的频道组 */
            pubsubPattern *pat = ln->value;

            /* 判断频道组是否和当前频道匹配，如果匹配，则发送通知信息 */
            if (stringmatchlen((char*)pat->pattern->ptr,
                                sdslen(pat->pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)) {
                addReply(pat->client,shared.mbulkhdr[4]);
                addReply(pat->client,shared.pmessagebulk);
                addReplyBulk(pat->client,pat->pattern);
                addReplyBulk(pat->client,channel);
                addReplyBulk(pat->client,message);
                receivers++;
            }
        }
        decrRefCount(channel);
    }
    return receivers;
}
```

可以看到，对于模式订阅，Redis会使用stringmatchlen函数进行正则匹配，如果匹配成功，说明该客户端关注的频道组中包含当前频道，那么就需要将消息发送给客户端

# 小结

本篇注意是对Redis订阅与发布功能的分析，源码比较简单，对于订阅功能，仅仅是将客户端和频道名(组)记录在某个数据结构中，当有其他客户端向某个频道执行发布功能时，检查数据结构中那些订阅了该频道的客户端，并向其发送消息

