---
title : Redis源码剖析（十二）有序集合跳表实现
date : 2018-01-20 11:16
categories : Redis
tags : Redis
---



有序集合是Redis对象系统中的一部分，其底层采用跳表和压缩列表两种形式存储，在上一篇介绍了跳表实现，就趁热打铁看一下有序集合的跳表实现

本篇主要涉及的是有序集合添加数据的命令，后面会看到，在命令的底层实现中，实际上还是调用跳表的接口

<!--more-->

# 存储结构

有序集合的定义在server.h文件中，不过除了跳表以外，有序集合又保存了一个字典，这个字典的作用是用来查找某个数据对应的分值。根据跳表的实现可知，跳表内部是采用分值排序的，通过分值查找数据还行，但是如果要通过数据查找分值，就显得力不从心了，所以Redis又维护了一个字典，用来完成通过数据查找分值的任务

```c
//server.h
/* 有序集合 */
typedef struct zset {
    dict *dict; /* 存储<数据，分值>键值对，用来通过数据查找分值 */
    zskiplist *zsl; /* 跳表，保存<分值，数据>，内部通过分值排序 */
} zset;
```

# 有序集合操作

## 添加数据

通过命令ZADD，可以实现向数据库中添加有序集合，该命令是由zaddGenericCommand函数实现的。函数中会先解析命令选项，命令参数，然后根据底层是由跳表实现还是由压缩列表实现执行不同的操作，都是调用二者的接口

```c
/* 向有序集合中添加数据 */
void zaddGenericCommand(client *c, int flags) {
    static char *nanerr = "resulting score is not a number (NaN)";
    /* 获取键 */
    robj *key = c->argv[1];
    robj *ele;
    robj *zobj;
    robj *curobj;
    double score = 0, *scores = NULL, curscore = 0.0;
    int j, elements;
    int scoreidx = 0;
	...
    scoreidx = 2;
    /* 获取参数选项，参数选项紧接在键的后面 */
	...
    /* 将参数选项转换为数值变量 */
    ...

    /* argc中保存所有参数个数，scoreidx保存第一个数据的分值位置
     * argc - scoreidx计算所有的分值，数据个数 */
    elements = c->argc-scoreidx;
    /* 由于分值和数据是成对出现的，这里判断输入的个数是否合法 */
    if (elements % 2) {
        addReply(c,shared.syntaxerr);
        return;
    }
    /* 除以2计算不同<分数，数据>对的个数 */
    elements /= 2; 

    /* 核查几个选项 */
    ...

    /* 为分值分配内存 */
    scores = zmalloc(sizeof(double)*elements);
    for (j = 0; j < elements; j++) {
        /* 将字符串类型转成double */
        if (getDoubleFromObjectOrReply(c,c->argv[scoreidx+j*2],&scores[j],NULL)
            != C_OK) goto cleanup;
    }
    /* 在数据库中查找是否存在键key，返回键对应的值 */
    zobj = lookupKeyWrite(c->db,key);
    if (zobj == NULL) {
        /* 如果不存在，创建值 */
        if (xx) goto reply_to_client; /* No key + XX option: nothing to do. */
        /* 根据参数配置选择底层采用压缩字典还是跳表 */
        /* 如果数据长度大于规定值，则采用跳表，否则选择压缩列表 */
        if (server.zset_max_ziplist_entries == 0 ||
            server.zset_max_ziplist_value < sdslen(c->argv[scoreidx+1]->ptr))
        {
            /* 创建跳表编码的有序集合 */
            zobj = createZsetObject();
        } else {
            /* 创建压缩列表编码的有序集合 */
            zobj = createZsetZiplistObject();
        }
        /* 将键值对添加到数据库中，这里值是空的 */
        dbAdd(c->db,key,zobj);
    } else {
        /* 存在键key，判断原先的值是否是用有序集合存储的 */
        if (zobj->type != OBJ_ZSET) {
            addReply(c,shared.wrongtypeerr);
            goto cleanup;
        }
    }

    /* 对于每个<分数，数据>对，将其添加到zobj中 */
    for (j = 0; j < elements; j++) {
        /* 第j个分值 */
        score = scores[j];
        /* 采用压缩列表的api执行添加操作 */
        if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
            ...
        } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
            /* 采用跳表的api添加数据 */
            zset *zs = zobj->ptr;
            zskiplistNode *znode;
            dictEntry *de;

            /* 尝试将数据转换成合适的编码以节省内存 */
            ele = c->argv[scoreidx+1+j*2] =
                tryObjectEncoding(c->argv[scoreidx+1+j*2]);
            /* 采用跳表实现的有序集合中保存了一个字典，键是数据，值是分值 */
            de = dictFind(zs->dict,ele);
            /* 有序集合中存在要添加的数据 */
            if (de != NULL) {
                if (nx) continue;
                /* 获取键节点的数据和分值 */
                curobj = dictGetKey(de);
                curscore = *(double*)dictGetVal(de);

                /* incr选项是如果存在该数据，则和它对应的分值相加 */
                if (incr) {
                    score += curscore;
                    ...
                }

                /* 更新数据，分值 */
                if (score != curscore) {
                    /* 先删除，再添加 */
                    serverAssertWithInfo(c,curobj,zslDelete(zs->zsl,curscore,curobj));
                    /* 跳表插入操作 */
                    znode = zslInsert(zs->zsl,score,curobj);
                    /* 增加数据的引用计数 */
                    incrRefCount(curobj); 
                    /* 更新字典中的分值 */
                    dictGetVal(de) = &znode->score; 
                    server.dirty++;
                    updated++;
                }
                processed++;
            } else if (!xx) {
                /* 不存在要添加的数据，直接插入 */
                znode = zslInsert(zs->zsl,score,ele);
                incrRefCount(ele); /* Inserted in skiplist. */
                serverAssertWithInfo(c,NULL,dictAdd(zs->dict,ele,&znode->score) == DICT_OK);
                incrRefCount(ele); /* Added to dictionary. */
                server.dirty++;
                added++;
                processed++;
            }
        } else {
            serverPanic("Unknown sorted set encoding");
        }
    }
	...
}
```

为了不让函数太长，这里删除了一些关于命令选项的判断和执行，不过还是很长...

## 获取排名

在跳表中，看到跳表可以快速计算<分值，数据>的排名。排名是由ZRANK命令完成的，底层由zrankGnericCommand函数实现

```c
//t_zset.c
/* 返回某个键下的指定数据的排名 */
void zrankGenericCommand(client *c, int reverse) {
    /* 第一个参数是键 */
    robj *key = c->argv[1];
    /* 第二个参数是值 */
    robj *ele = c->argv[2];
    robj *zobj;
    unsigned long llen;
    unsigned long rank;

    /* 在数据库中查找键key是否存在，如果存在，再判断值是否是由有序集合存储的 */
    if ((zobj = lookupKeyReadOrReply(c,key,shared.nullbulk)) == NULL ||
        checkType(c,zobj,OBJ_ZSET)) return;
    /* 获取有序集合中数据个数 */
    llen = zsetLength(zobj);

    serverAssertWithInfo(c,ele,sdsEncodedObject(ele));

    /* 根据底层实现不同选择不同的接口 */
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        ...
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        /* 如果底层采用跳表实现，则调用跳表接口 */
        /* 获取键key对应的有序集合 */
        zset *zs = zobj->ptr;
        zskiplist *zsl = zs->zsl;
        dictEntry *de;
        double score;

        ele = c->argv[2];
        /* 由于跳表通过数据查找分值比较慢
         * 所以Redis采用字典保存<数据，分值>对，可通过数据快速找到对应分值 */
        de = dictFind(zs->dict,ele);
        if (de != NULL) {
            /* 如果有序集合中存在要查找的数据，则获取数据的分值 */
            score = *(double*)dictGetVal(de);
            /* 调用跳表接口计算分值score的排名 */
            rank = zslGetRank(zsl,score,ele);
            serverAssertWithInfo(c,ele,rank); 
            /* 根据选项不同计算是正向排名还是逆向排名 */
            if (reverse)
                addReplyLongLong(c,llen-rank);
            else
                addReplyLongLong(c,rank-1);
        } else {
            addReply(c,shared.nullbulk);
        }
    } else {
        serverPanic("Unknown sorted set encoding");
    }
}
```

## 计算数据个数

zsetLength函数用于计算有序集合中数据个数，同样是调用跳表或者压缩列表的接口

```c
//t_zset.c
/* 计算有序集合中数据个数 */
unsigned int zsetLength(robj *zobj) {
    int length = -1;
    /* 根据底层实现不同调用不同接口 */
    if (zobj->encoding == OBJ_ENCODING_ZIPLIST) {
        ...
    } else if (zobj->encoding == OBJ_ENCODING_SKIPLIST) {
        /* 跳表中直接保存了数据个数 */
        length = ((zset*)zobj->ptr)->zsl->length;
    } else {
        serverPanic("Unknown sorted set encoding");
    }
    return length;
}
```

# 小结

有序集合中保存的数据都是有序的，在对象系统中底层可以由跳表和有序列表实现，而且跳表实现还是比较简单的，而压缩列表实现会相对难理解一些