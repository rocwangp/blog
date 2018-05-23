---
title : Redis源码剖析（九）对象系统概述
date : 2018-01-09 20:28
categories : Redis
tags : Redis
---



在Redis的源码中，到处可见robj类型的变量，在介绍其他模块时，只是将它看成Redis的数据类型，并没有深入探究。而事实上，它是对象系统，提供了对多种类型的封装，Redis可以根据数据的具体形式，采用不同的类型进行存储，一方面提高了灵活性，一方面也为节省内存提供了便利，因为Redis所有的数据都是直接存在内存中的，所以需要想方设法节省内存

<!--more-->

# 对象结构

redisObject结构中包含了对象系统的定义，记录了数据类型，数据编码格式，最后一次访问的时间，引用计数，值

```c
//server.h
/* 对象系统的定义 */
typedef struct redisObject {
    unsigned type:4; //类型，可以是string, hash, list, set和zset(宏定义给出)
    unsigned encoding:4; //编码,表示ptr底层数据以何种方式存储
    unsigned lru:LRU_BITS; //最后一次访问的时间
    int refcount; //引用计数
    void *ptr; //实际存放的值
} robj;
```

>:n是位域，显式指出该变量占用的位数，上述定义中，type占4位，encoding占4位，二者共占8位，即1个字节

## 类型

类型就是命令指出的数据格式

|  命令   |        操作        |                    举例                    |
| :---: | :--------------: | :--------------------------------------: |
|  SET  | 键为字符串对象，值为字符串对象  |               SET db redis               |
| SADD  |  键为字符串对象，值为集合对象  |       SADD db redis mongodb mysql        |
| RPUSH |  键为字符串对象，值为列表对象  |       RPUSH db redis mongodb mysql       |
| HMSET |  键为字符串对象，值为哈希对象  |  HMSET profile name Tom age 25 sex male  |
| ZADD  | 键为字符串对象，值为有序集合对象 | ZADD price 8.5 apple 5.0 banana 6.0 cherry |

这5种类型由宏定义给出

```c
//server.h
#define OBJ_STRING 0
#define OBJ_LIST 1
#define OBJ_SET 2
#define OBJ_ZSET 3
#define OBJ_HASH 4
```
Redis提供了TYPE命令用于返回不同数据的类型

```c
127.0.0.1:6379> SET db redis
OK
127.0.0.1:6379> TYPE db	//SET，字符串类型值
string
127.0.0.1:6379> SADD db_sadd redis mongodb mysql
(integer) 3
127.0.0.1:6379> TYPE db_sadd	//SADD，集合类型值
set
127.0.0.1:6379> ZADD price 8.5 apple 5.0 banana 6.0 cherry
(integer) 3
127.0.0.1:6379> TYPE price	//ZADD，有序集合类型值
zset
127.0.0.1:6379> RPUSH db_rpush redis mongodb mysql
(integer) 3
127.0.0.1:6379> TYPE db_rpush	//RPUSH，列表类型值
list
127.0.0.1:6379> HMSET profile name Tom age 25 sex male
OK
127.0.0.1:6379> TYPE profile	//HMSET，哈希表类型值
hash
```
## 编码

编码代表数据实际的存储格式，实际保存的类型和提供的类型不一定相同，举个例子，如果使用SET version 10添加一个字符串类型的键值对<version, 10>，那么在Redis内部，10是采用整型编码存储的而非提供的字符串，因为对于数字而言，整型在某些时候需要的字节要少一些，同样由宏定义给出所有编码格式

```c
#define OBJ_ENCODING_RAW 0     /* Raw格式，常规字符串类型 */
#define OBJ_ENCODING_INT 1     /* 整数形式 */
#define OBJ_ENCODING_HT 2      /* 哈希表 */
#define OBJ_ENCODING_ZIPMAP 3  /* 压缩字典 */
#define OBJ_ENCODING_LINKEDLIST 4 /* 双端链表 */
#define OBJ_ENCODING_ZIPLIST 5 /* 压缩列表 */
#define OBJ_ENCODING_INTSET 6  /* 整数集合 */
#define OBJ_ENCODING_SKIPLIST 7  /* 跳表 */
#define OBJ_ENCODING_EMBSTR 8  /* EMBSTR格式，适用于存储较短的字符串类型，比Raw少申请一次内存 */
#define OBJ_ENCODING_QUICKLIST 9 /* 快速列表 */
```

Redis提供OBJECT ENCODING命令获取键对应的值在底层的编码格式

|   底层数据结构    |          编码常亮           | OBJECT ENCODING命令输出 |
| :---------: | :---------------------: | :-----------------: |
|     整数      |    OBJ_ENCODING_INT     |        "int"        |
| embstr编码字符串 |   OBJ_ENCODING_EMBSTR   |      "embstr"       |
|  raw编码字符串   |    OBJ_ENCODING_RAW     |        "raw"        |
|     字典      |     OBJ_ENCODING_HT     |     "hashtable"     |
|    双端链表     | OBJ_ENCODING_LINKEDLIST |    "linkedlist"     |
|    压缩列表     |  OBJ_ENCODING_ZIPLIST   |      "ziplist"      |
|    整数集和     |   OBJ_ENCODING_INTSET   |      "intset"       |
|     跳表      |  OBJ_ENCODING_SKIPLIST  |     "skiplist"      |

```c
127.0.0.1:6379> set db_set_embstr redis
OK
127.0.0.1:6379> OBJECT ENCODING db_set_embstr	//短字符串采用embstr编码
"embstr"
127.0.0.1:6379> set db_set_raw "long long long long long long long long long long ago ..."
OK
127.0.0.1:6379> OBJECT ENCODING db_set_raw	//长字符串采用raw编码
"raw"
127.0.0.1:6379> SADD numbers 1 3 5	
(integer) 3
127.0.0.1:6379> OBJECT ENCODING numbers	//只有数字，采用整数集合
"intset"
127.0.0.1:6379> SADD numbers "seven"
(integer) 1
127.0.0.1:6379> OBJECT ENCODING numbers	//增加了一个字符串，不能再采用整数集合，改为哈希表
"hashtable"
```

可以看到，Redis会自适应改变数据底层的编码格式，而不是固定和一种类型绑定，这大大提高了灵活性



## 最后一次访问时间

用来记录最后一次访问该数据的时间，可以获得该数据的空转时长，使用频率等

## 引用计数

模仿C++的智能指针，使多个对象共享同一个底层数据，以便于节省内存占用，当引用计数为0时，Redis会释放该对象的内存。



# 对象创建

对象操作主要涉及根据不同类型创建不同对象等操作

最基本的创建对象操作由createObject函数完成，函数根据给定类型和值创建编码格式为raw的对象，其它创建对象的函数大多数都是直接或间接调用该函数

```c
//object.c
/* 根据type和ptr创建编码为raw字符串的对象 */
robj *createObject(int type, void *ptr) {
    /* 申请对象内存空间 */
    robj *o = zmalloc(sizeof(*o));
    /* 设置类型，编码，值，引用计数初始化为1 */
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution). */
    /* 计算当前时间，赋值给lru作为最后一次访问时间 */
    o->lru = LRU_CLOCK();
    /* 返回对象指针 */
    return o;
}
```

## 创建字符串类型对象

字符串有raw和embstr两种类型和编码格式，raw适用于长字符串，需要执行两次动态内存的申请，而embstr适用于短字符串，仅仅需要一次内存申请，在创建字符串类型的对象时，Redis会判断数据的长度以决定采用哪一个

```c
//object.c
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
/* 创建字符串类型对象，根据数据长度不同选择不同的类型格式 */
robj *createStringObject(const char *ptr, size_t len) {
    /* 根据长度不同选择不同的编码方式 */
    if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}
```

可以看到，长度小于44的字符串默认都采用embstr，而大于44的采用raw

raw类型的字符串对象创建直接调用createObject函数即可，因为raw类型的字符串底层编码也是raw

```c
//object.c
/* 创建raw字符串类型变量 */
robj *createRawStringObject(const char *ptr, size_t len) {
    /* sdsnewlen()创建一个长度为len的sds字符串 */
    return createObject(OBJ_STRING,sdsnewlen(ptr,len));
}
```

>sdsnewlen函数是创建一个长度为len，值为ptr的sds变量

embstr类型的字符串创建不可以调用createObject函数，由于采用embstr编码格式，数据分布是不同的，需要重新实现创建函数

```c
//object.c
/* 创建类型为embstr，编码为embstr的字符串对象 */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    /* 和sds对象的创建有关 */
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    struct sdshdr8 *sh = (void*)(o+1);

    /* 设置类型，编码，数据，引用计数，最后一次访问时间 */
    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    o->lru = LRU_CLOCK();

    /* 将数据复制给sds对象，和sds有关 */
    sh->len = len;
    sh->alloc = len;
    sh->flags = SDS_TYPE_8;
    if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}
```

此外，Redis还提供根据长整型，长浮点型创建一个字符串类型对象，本质都一样，这里不再一一赘述

## 创建其它类型对象

除了字符串类型对象之外，其它类型对象的创建都显得比较简单，仅仅是创建一个相应类型的变量，然后调用createObject函数，返回后将编码格式改成对应类型的编码格式

```c
//object.c
/* 创建快速列表对象 */
robj *createQuicklistObject(void) {
    quicklist *l = quicklistCreate();
    robj *o = createObject(OBJ_LIST,l);
    o->encoding = OBJ_ENCODING_QUICKLIST;
    return o;
}

/* 创建压缩列表对象 */
robj *createZiplistObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_LIST,zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}

/* 创建集合对象 */
robj *createSetObject(void) {
    dict *d = dictCreate(&setDictType,NULL);
    robj *o = createObject(OBJ_SET,d);
    o->encoding = OBJ_ENCODING_HT;
    return o;
}

/* 创建整数集合对象 */
robj *createIntsetObject(void) {
    intset *is = intsetNew();
    robj *o = createObject(OBJ_SET,is);
    o->encoding = OBJ_ENCODING_INTSET;
    return o;
}

/* 创建哈希对象 */
robj *createHashObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_HASH, zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}

/* 创建有序集合对象 */
robj *createZsetObject(void) {
    zset *zs = zmalloc(sizeof(*zs));
    robj *o;

    zs->dict = dictCreate(&zsetDictType,NULL);
    zs->zsl = zslCreate();
    o = createObject(OBJ_ZSET,zs);
    o->encoding = OBJ_ENCODING_SKIPLIST;
    return o;
}

/* 创建集合压缩列表对象 */
robj *createZsetZiplistObject(void) {
    unsigned char *zl = ziplistNew();
    robj *o = createObject(OBJ_ZSET,zl);
    o->encoding = OBJ_ENCODING_ZIPLIST;
    return o;
}
```



# 小结

本篇主要是对Redis对象系统的一个概述，核心目的就是弄清楚Redis底层的类型和编码都有哪些，接下来会对每个数据结构进行具体的分析，到时候还会引用本篇的部分代码。分析数据结构是最无聊的事情，也正因为如此才没有在最开始分析，不过为了后面的持久化功能，对象系统是不得不啃的骨头，希望自己能够坚持！