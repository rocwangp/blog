---
title : Redis源码剖析（八）链表
date : 2018-01-09 18:00
categories : Redis
tags : Redis
---



在之前对Redis的介绍中，可以看到链表的使用频率非常高。

链表可以作为单独的存储结构，比如客户端的监视链表记录该客户端监视的所有键，服务器的模式订阅链表记录所有客户端和它的模式订阅。

链表也可以内嵌到字典中作为字典的值类型，比如数据库的监视字典使用链表存储监视某个键的所有客户端，服务器的订阅字典使用链表存储订阅某个频道的所有客户端。

<!--more-->

# 链表结构

## 节点

Redis中的链表是双向链表，即每一个节点都保存了它的前驱节点和后继节点，用于提高操作效率。节点定义如下

```c
//adlist.h
/* 链表节点 */
typedef struct listNode {
    struct listNode *prev; /* 前驱节点 */
    struct listNode *next; /* 后继节点 */
    void *value;    /* 值 */
} listNode;
```

## 链表

链表结构主要记录了表头节点和表尾节点，节点个数以及一些函数指针，定义如下

```c
//adlist.h
/* 链表 */
typedef struct list {
    listNode *head; /* 链表头节点 */
    listNode *tail; /* 链表尾节点 */
    void *(*dup)(void *ptr); /* 节点值复制函数 */
    void (*free)(void *ptr); /* 节点值析构函数 */
    int (*match)(void *ptr, void *key); /* 节点值匹配函数 */
    unsigned long len; /* 链表长度 */
} list;
```

函数指针主要是对节点值的操作，包括复制，析构，判断是否相等

## 迭代器

此外，Redis还为链表提供迭代器的功能，主要是对链表节点的封装，另外通过链表节点的前驱节点和后继节点，可以轻松的完成向前移动和向后移动

```c
//adlist.h
/* 迭代器 */
typedef struct listIter {
    /* 指向实际的节点 */
    listNode *next;
    /* 迭代器方向，向前还是向后 */
    int direction;
} listIter;
```

direction的值有两个，向前和向后，由宏定义指出

```c
//adlist.h
#define AL_START_HEAD 0	/* 从头到尾(向后) */
#define AL_START_TAIL 1 /* 从尾到头(向前) */
```

# 链表操作

## 创建链表

链表的创建工作由listCreate函数完成，实际上就是申请链表内存然后初始化成员变量

```c
//adlist.c
/* 创建一个空链表 */
list *listCreate(void)
{
    struct list *list;

    /* 为链表申请内存 */
    if ((list = zmalloc(sizeof(*list))) == NULL)
        return NULL;
    /* 初始化 */
    list->head = list->tail = NULL;
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    return list;
}
```

## 删除链表

删除一个链表比创建稍微麻烦一点，因为需要释放每个节点中保存的值，没错，它正是调用free函数完成的

```c
//adlist.c
/* 释放链表的内存空间 */
void listRelease(list *list)
{
    unsigned long len;
    listNode *current, *next;

    current = list->head;
    len = list->len;
    /* 遍历链表，释放每一个节点 */
    while(len--) {
        /* 记录下一个节点 */
        next = current->next;
        /* 如果定义了节点值析构函数，则调用 */
        if (list->free) list->free(current->value);
        /* 释放节点内存 */
        zfree(current);
        current = next;
    }
    /* 因为list* 也是动态申请的，所以也需要释放 */
    zfree(list);
}
```

## 在末尾插入节点

在其他模块的实现上，经常会看到向链表尾部添加节点的操作，它的实现由listAddNodeTail完成。函数首先为新节点申请内存，然后将节点添加到链表中，这里需要根据链表之前是否为空执行不同操作

* 链表为空，新节点将作为链表的头节点和尾节点，新节点的前驱和后继指针都为空
* 链表非空，新节点将作为链表的尾节点，之前的尾节点的后继指针指向新节点，新节点的前驱指针指向之前的尾节点

```c
//adlist.c
/* 在链表尾部添加节点 */
list *listAddNodeTail(list *list, void *value)
{
    listNode *node;

    /* 申请节点 */
    if ((node = zmalloc(sizeof(*node))) == NULL)
        return NULL;
    /* 记录节点值 */
    node->value = value;
    /* 如果之前链表为空，那么插入一个节点后头尾节点都是新节点 */
    if (list->len == 0) {
        list->head = list->tail = node;
        /* 设置前驱后继节点 */
        node->prev = node->next = NULL;
    } else {
        /* 不为空，只改变尾节点 */
        node->prev = list->tail;
        node->next = NULL;
        list->tail->next = node;
        list->tail = node;
    }
    /* 节点个数加一 */
    list->len++;
    return list;
}
```

## 迭代器移动

迭代器主要用于遍历链表，而迭代器的重点在移动上，通过direction变量，可以得知迭代器移动的方向，又通过链表节点的前驱后继节点，可以轻松实现移动操作

```c
//adlist.c
/* 移动迭代器，同时返回下一个节点 */
listNode *listNext(listIter *iter)
{
    /* next指针是当前迭代器指向的节点指针 */
    listNode *current = iter->next;

    if (current != NULL) {
        /* 根据方向为next赋值 */
        if (iter->direction == AL_START_HEAD)
            iter->next = current->next;
        else
            iter->next = current->prev;
    }
    /* 返回之前迭代器指向的节点 */
    return current;
}
```

## 重置迭代器

此外，Redis提供了重置迭代器的操作，分别由listRewind和listRewindTail函数完成

```c
/* 重置迭代器方向为从头到尾，使迭代器指向头节点 */
void listRewind(list *list, listIter *li) {
    li->next = list->head;
    li->direction = AL_START_HEAD;
}

/* 重置迭代器方向为从尾到头，使迭代器指向尾节点 */
void listRewindTail(list *list, listIter *li) {
    li->next = list->tail;
    li->direction = AL_START_TAIL;
}
```

## 链表搜索

有了迭代器的基础，就可以实现链表搜索功能，即在链表中查找与某个值匹配的节点，需要利用迭代器遍历链表

```c
//adlist.c
/* 查找值key，返回链表节点 */
listNode *listSearchKey(list *list, void *key)
{
    listIter iter;
    listNode *node;

    /* 设置迭代器方向为从头到尾，使其指向链表头节点 */
    listRewind(list, &iter);
    /* 遍历链表 */
    while((node = listNext(&iter)) != NULL) {
        /* 如果提供值匹配函数，则调用，否则使用==比较 */
        if (list->match) {
            if (list->match(node->value, key)) {
                return node;
            }
        } else {
            if (key == node->value) {
                return node;
            }
        }
    }
    return NULL;
}
```

## 宏定义函数

除了上面提到的函数外，Redis还提供了一些宏定义函数，比如返回节点值，返回节点的前驱后继节点等

```c
//adlist.h
/* 返回链表节点个数 */
#define listLength(l) ((l)->len)
/* 返回头节点 */
#define listFirst(l) ((l)->head)
/* 返回尾节点 */
#define listLast(l) ((l)->tail)
/* 返回前驱节点 */
#define listPrevNode(n) ((n)->prev)
/* 返回后继节点 */
#define listNextNode(n) ((n)->next)
/* 返回节点值 */
#define listNodeValue(n) ((n)->value)

/* 设置链表的值复制，值析构，值匹配函数 */
#define listSetDupMethod(l,m) ((l)->dup = (m))
#define listSetFreeMethod(l,m) ((l)->free = (m))
#define listSetMatchMethod(l,m) ((l)->match = (m))

/* 获取链表的值赋值，值析构，值匹配函数 */
#define listGetDupMethod(l) ((l)->dup)
#define listGetFree(l) ((l)->free)
#define listGetMatchMethod(l) ((l)->match)
```



# 小结

由于链表结构简单，所以在实现上还是非常容易理解的。当然Redis中与链表有关的函数还有很多很多，这里仅仅介绍了一些常用操作，有兴趣可以深入源码查看