---
title : Redis源码剖析（四）过期键的删除策略
date : 2018-01-04 22:00
categories : Redis
tags : Redis
---



Redis是支持时间事件的，所谓时间事件，是为某个键值对设置过期时间，时间一到，Redis会自动删除该键值对。例如使用SET命令添加字符串类型的键值对

```c
127.0.0.1:6379> SET blog redis ex 10	//添加键值对<blog, redis>，10秒后删除
OK
127.0.0.1:6379> GET blog	//添加后马上查找，可以获取redis
"redis"
127.0.0.1:6379> GET blog	//上趟厕所回来，发现找不到了
(nil)
```

Redis是如何实现定时删除的呢，在数据库结构redisDb中，可以发现除了上篇提到的用于保存键值对的dict字典外，另有一个字典变量expires，实际上正是它保存着键和其过期时间(绝对时间)。当执行完SET命令后，两个字典的数据分布为

<!--more-->

```c
//server.h
typedef struct redisDb {
    dict *dict;            /* 保存键值对的字典 */     
    dict *expires;         /* 保存键和过期时间 */    
    int id;        /* 数据库唯一id */           
 	...
} redisDb;
```



```c
dict字典
blog --> redis
expires字典
blog --> blog的过期时间
```



# 设置，获取，删除过期时间

> 以下键节点指字典中的哈希节点，保存键和值

## 设置键的过期时间

```c
//db.c
/* 
 * 设置键的过期时间
 * db   : 数据库
 * key  : 键
 * when : 过期时间(绝对时间)
 */
void setExpire(redisDb *db, robj *key, long long when) {
    dictEntry *kde, *de;

    /* Reuse the sds from the main dict in the expire dict */
    /* 从数据字典中寻找键节点 */
    kde = dictFind(db->dict,key->ptr);
    serverAssertWithInfo(NULL,key,kde != NULL);
    /* 从时间字典中寻找键节点，如果不存在则创建一个 */
    de = dictReplaceRaw(db->expires,dictGetKey(kde));
    /* 设置键节点的值，值为过期时间(绝对时间) */
    dictSetSignedIntegerVal(de,when);
}
```

dictSetSignedIntegerVal是宏定义，设置键节点de的值为when。因为哈希节点中的值结构是联合，可以存储不同大小的数字，也可以通过void*指针存储其它类型，这里过期时间是long long类型，所以可以存在int64_t类型上

```c
//dict.h
#define dictSetSignedIntegerVal(entry, _val_) \
    do { entry->v.s64 = _val_; } while(0)
```



## 获取键的过期时间

```c
//db.c
/* 返回键的过期时间 */
long long getExpire(redisDb *db, robj *key) {
    dictEntry *de;

    /* 从时间字典中查找匹配的键节点 */
    if (dictSize(db->expires) == 0 ||
       (de = dictFind(db->expires,key->ptr)) == NULL) return -1;
       
    serverAssertWithInfo(NULL,key,dictFind(db->dict,key->ptr) != NULL);
    /* 返回键节点对应的值 */
    return dictGetSignedIntegerVal(de);
}
```

## 删除键的过期时间

```c
//db.c
/* 移除键的过期时间 */
int removeExpire(redisDb *db, robj *key) {
    serverAssertWithInfo(NULL,key,dictFind(db->dict,key->ptr) != NULL);
    /* 从时间字典中将键删除 */
    return dictDelete(db->expires,key->ptr) == DICT_OK;
}
```

上面三个函数都是调用字典dict的接口，比较好理解



# 过期键删除策略

对于过期键值对的删除有三种策略，分别是

- 定时删除，设置一个定时器和回调函数，时间一到就调用回调函数删除键值对。优点是及时删除，缺点是需要为每个键值对都设置定时器，比较麻烦(其实可以用timer_fd的，参考muduo定时任务的实现)
- 惰性删除，只有当再次访问该键时才判断是否过期，如果过期将其删除。优点是不需要为每个键值对进行时间监听，缺点是如果这个键值对一直不被访问，那么即使过期也会一直残留在数据库中，占用不必要的内存
- 周期删除，每隔一段时间执行一次删除过期键值对的操作。优点是既不需要监听每个键值对导致占用CPU，也不会一直不删除导致占用内存，缺点是不容易确定删除操作的执行时长和频率

Redis采用惰性删除和周期删除两种策略，通过配合使用，服务器可以很好的合理使用CPU时间和避免内不能空间的浪费

## 惰性删除

惰性删除是指在对每一个键进行读写操作时，先判断一下这个键是否已经过期，如果过期则将其删除。该操作由expireIfNeeded函数完成

```c
//db.c
/* 判断键key是否已过期，如果过期将其从数据库中删除 */
int expireIfNeeded(redisDb *db, robj *key) {
    /* 获取键的过期时间*/
    mstime_t when = getExpire(db,key);
    mstime_t now;

    /* 该键没有设置过期时间 */
    if (when < 0) return 0; /* No expire for this key */

    /* Don't expire anything while loading. It will be done later. */
    if (server.loading) return 0;

    /* 获取当前时间，lua脚本相关 */
    now = server.lua_caller ? server.lua_time_start : mstime();

    if (server.masterhost != NULL) return now > when;

    /* 当前时间小于过期时间，该键没有过期，不需要删除 */
    if (now <= when) return 0;
    
    /* 执行到这里，说明这个键已过期，需要删除 */
    /* 过期键数量加一 */
    server.stat_expiredkeys++;
    propagateExpire(db,key);
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    /* 从数据字典和时间字典中删除(即从数据库中删除，因为该键在两个字典中，所以需要删除两个) */
    return dbDelete(db,key);
}
```

expireIfNeeded函数只是判断是否需要删除键节点，实际删除操作由dbDelete函数完成

该函数调用字典的删除接口完成删除操作，该接口在上一篇有提到过

```c
//db.c
/* 将键key从数据库中删除 */
int dbDelete(redisDb *db, robj *key) {
    /* Deleting an entry from the expires dict will not free the sds of
     * the key, because it is shared with the main dictionary. */
    /* 从时间字典中删除 */
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
    /* 从数据字典中删除 */
    if (dictDelete(db->dict,key->ptr) == DICT_OK) {
        /* 集群相关 */
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
```



## 周期删除

Redis服务器会周期性地执行server.c/serverCron函数，在这个函数中执行的databasesCron函数会调用activeExpireCycle函数，这个函数在时间字典(expires)中随机选择若干键节点，判断其是否过期，如果过期则将其删除

```c
//server.c
/* 随机选择若干键节点判断其是否过期，如果过期将其删除 */
void activeExpireCycle(int type) {
	...
    while (num--) {
    	dictEntry *de;
        long long ttl;

     	/* 在时间字典中随机选择一个键节点 */
      	if ((de = dictGetRandomKey(db->expires)) == NULL) break;
      	/* 获取键节点的值，即过期时间 */
      	ttl = dictGetSignedIntegerVal(de)-now;
      	/* 判断是否过期，如果过期，删除 */
      	if (activeExpireCycleTryExpire(db,de,now)) expired++;
      	if (ttl > 0) {
      	/* We want the average TTL of keys yet not expired. */
      		ttl_sum += ttl;
      		ttl_samples++;
      	}
	}
	...
}
```

随机选择键节点后，调用activeExpireCycleTryExpire判断其是否过期

```c
//server.c
/* 
 * 判断键节点是否过期，如果过期将其从数据库中删除 
 * db  : 数据库
 * de  : 键节点
 * now : 当前时间
 */
int activeExpireCycleTryExpire(redisDb *db, dictEntry *de, long long now) {
    /* 获取键的过期时间 */
    long long t = dictGetSignedIntegerVal(de);
    /* 判断是否过期 */
    if (now > t) {
        /* 从键节点中取出键 */
        sds key = dictGetKey(de);
        /* 因为Redis中键默认都是sds存储的，所以这里需要将其转化为robj*格式以满足函数传参 */
        robj *keyobj = createStringObject(key,sdslen(key));

        propagateExpire(db,keyobj);
        /* 将键和其对应的值从数据库中删除 */
        dbDelete(db,keyobj);
        notifyKeyspaceEvent(NOTIFY_EXPIRED,
            "expired",keyobj,db->id);
        /* 键的引用计数减一，因为是刚创建的，所以引用计数就是1，这里会将keyobj对象删除 */
        decrRefCount(keyobj);
        /* 过期键个数加一 */
        server.stat_expiredkeys++;
        return 1;
    } else {
        return 0;
    }
}
```

## 随机选择键节点

随机选择键节点是字典的接口，该函数利用随机函数选择一个下标，确保当前下标下存在键节点后进行第二次随机，选择该下标下的某个键节点返回，函数定义如下

```c
/* 随机返回一个键节点 */
dictEntry *dictGetRandomKey(dict *d)
{
    dictEntry *he, *orighe;
    unsigned int h;
    int listlen, listele;

    /* 字典中没有键节点，返回 */
    if (dictSize(d) == 0) return NULL;
    /* 处于rehash状态，执行一步rehash */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    /* 如果处于rehash状态，那么随机操作在ht[0]和ht[1]两个字典中进行 */
    /* 随机选择一个下标，该下标下存在键节点 */
    if (dictIsRehashing(d)) {
        do {
            h = d->rehashidx + (random() % (d->ht[0].size +
                                            d->ht[1].size -
                                            d->rehashidx));
            he = (h >= d->ht[0].size) ? d->ht[1].table[h - d->ht[0].size] :
                                      d->ht[0].table[h];
        } while(he == NULL);
    } else {
        do {
            h = random() & d->ht[0].sizemask;
            he = d->ht[0].table[h];
        } while(he == NULL);
    }

    listlen = 0;
    orighe = he;
    /* 计算随机的下标下的键节点个数 */
    while(he) {
        he = he->next;
        listlen++;
    }
    /* 随机选择一个键节点返回 */
    listele = random() % listlen;
    he = orighe;
    while(listele--) he = he->next;
    return he;
}
```



# 小结

Redis允许为每个键值对设置过期时间，时间一到会将其从数据库中删除。Redis内部采用惰性删除和周期删除两种策略结合的方法删除过期键，其中惰性删除是指当访问键时才判断该键是否过期，周期删除是每隔一定时间进行一次集中删除，一次集中删除随机判断一定数量的键是否过期并将过期键删除