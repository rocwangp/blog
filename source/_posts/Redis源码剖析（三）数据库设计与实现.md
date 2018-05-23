---
title : Redis源码剖析（三）字典结构的设计与实现
date : 2018-01-04 18:50
categories : Redis
tags : Redis
---



Redis是K-V类型的数据库，所谓K-V类型，就是底层存储的数据结构是key-value，即键key，值value。键key在Redis中以字符串的形式存在，而值value可以是多种类型

Redis内部的键值对采用字典存储，而字典底层又采用哈希表实现。哈希表是常用的键值对存储结构，根据键key计算哈希值，然后计算索引下标，在哈希表中对应下标处存储键key对应的值。因为不同key被映射到同一个下标是很常见的事情，所以哈希表一般都需要解决冲突，Redis使用开链法解决冲突，即哈希表每个下标处都是一个链表，用来保存被映射到相同下标的键值对

<!--more-->

# 数据库结构

Redis的数据库结构定义在server.h中，内部包含多个字典，分别适用于不同的场景。因为还没有涉及到Redis其他功能的实现，所以这里只关心保存键值对的字典

```c
//server.h
typedef struct redisDb {
    dict *dict;    /* 保存键值对的字典 */    
    int id;        /* 数据库唯一id */   
    ...
} redisDb; 
```

dict变量是是存储键值对数据的字典，在进行数据库的曾，删，改，查时都需要使用这个字典。下面就来看一下字典在底层是如何实现的

# 字典的设计与实现

想要实现字典，首先需要实现哈希表，由于采用开链法解决冲突问题，就需要先设计哈希节点，所以在dict.h中可以清楚的看到对于这些结构的定义

## 哈希节点

其中，哈希节点的定义如下，由于采用开链法，所以哈希表每个元素都应该是个链表。对于哈希节点的值，采用联合结构是为了可以保存不同类型的值，对于非数值类型的数据，可以通过void*指针保存。后面可以看到，Redis的对象系统将所有数据类型进行了统一

```c
//dict.h
/* 哈希表节点，使用开链法解决冲突 */
typedef struct dictEntry {
    void *key; //键
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v; //值，可以保存不同数值类型，而其它的类型可以通过void*指针保存
    struct dictEntry *next; /* 下一个哈希节点 */
} dictEntry;
```

## 哈希表

接下来是哈希表的定义，哈希表主要保存的就是一个大数组，数组每个元素都是dictEntry*，指针表示的实际是哈希节点，所以可以看到每个数组元素都是一个链表

```c
//dict.h
/* 哈希表 */
typedef struct dictht {
    dictEntry **table; //哈希表数组，数组元素是链表(开链法解决冲突)
    unsigned long size; //哈希表大小
    unsigned long sizemask; //掩码，用于计算索引，总是等于size-1
    unsigned long used; //已有节点数量
} dictht;
```

这里需要区分一下size和used的区别，size是指哈希表的大小，即数组大小，used是指哈希表中哈希节点的个数，一个下标下可能会有多个哈希节点。后面会看到，size和used的大小关系决定了是否需要对哈希表进行扩充或收缩(rehash操作)

sizemask的掩码，用来计算键key对应的下标。通常通过哈希算法计算的哈希值会大于哈希表大小，通过sizemask，可以将哈希值的大小映射到[0:size-1]范围内，采用的方法是将哈希值与sizemask做与运算，实际上就是对size取模 



## 字典

Redis的字典主要保存两个哈希表(以下简称为ht[0]和ht[1])，ht[0]是实际保存键值对的哈希表，ht[1]只有当进行rehash时才会用到

rehashidx是rehash的进度，后面会看到用处

```c
//dict.h
/* 字典 */
typedef struct dict {
    dictType *type; /* 字典处理函数 */
    void *privdata; //用于传给函数的参数
    dictht ht[2]; //两个哈希表，多出的一个用于rehash时用
    long rehashidx; /* rehash时使用，记录rehash的进度 */
    int iterators; 
} dict;
```



dictType类型定义如下，目前可以简单理解成是字典处理函数，根据不同类型的键值对提供不同的处理函数

根据函数名可以很好理解函数用途，需要注意第一个函数hashFunction是计算哈希值的函数

```c
//dict.h
/* 哈希表处理函数 */
typedef struct dictType {
    unsigned int (*hashFunction)(const void *key); // 计算哈希值
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

# 字典的Rehash操作

本来是想在最后介绍rehash的，结果发现在字典的基本操作中处处涉及到rehash，所以想想还是将rehash挪到前面好了

Redis会在两种情况对字典进行rehash操作

- 字典中哈希节点数量过多，导致在同一个下标下查找某个键的时间边长，需要扩大字典容量
- 字典中哈希节点数量过少，导致很多位置处于空缺，占用大量空缺内存，需要缩小字典容量

## 哈希表的扩充

当为键计算哈希值时，会判断是否需要对字典进行扩充，判断函数是_dictExpandIfNeeded，判断的依据是

- 字典中哈希节点个数大于哈希表大小
- 哈希节点个数 / 哈希表大小 > 负载因子

函数定义如下

```c
//dict.c
/* 判断是否需要对字典执行扩充操作，如果需要，扩充 */
static int _dictExpandIfNeeded(dict *d)
{
    /* 如果此时正在进行rehash操作，则不进行扩充 */
    if (dictIsRehashing(d)) return DICT_OK;

    /* 如果哈希表容量为0，代表此时是初始化，创建默认大小的哈希表 */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* 负载因子，如果字典中的节点数量 / 字典大小 > 负载因子，就需要对字典进行扩充 */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```



rehash的意思是进行重新映射，此时就需要使用ht[1]，即第二个哈希表(字典中包含两个哈希表，第二个用于rehash操作，上面提到过)

rehash的方法是先确定扩充或缩小的哈希表大小，然后为ht[1]申请内存空间，最后将rehashidx设置成0，标志着开始执行rehash操作

这些操作在dictExpand函数中执行

```c
//dict.c
/* 扩充字典的哈希表，扩充的主要目的是用于rehash  */
int dictExpand(dict *d, unsigned long size)
{
    /* 新的哈希表 */
    dictht n; /* the new hash table */
    /* _dictNextPower函数返回第一个大于size的2的幂次方，Redis保证哈希表的大小永远是2的幂次方 */
    unsigned long realsize = _dictNextPower(size);

    /* rehash的过程中不能进行扩充操作 */
    /* 扩充后的大小小于原有哈希节点时报错,说明size太小 */
    if (dictIsRehashing(d) || d->ht[0].used > size)
        return DICT_ERR;

    /* 大小没有改变,返回 */
    if (realsize == d->ht[0].size) return DICT_ERR;

    /* 设置新的哈希表属性，分配空间 */
    n.size = realsize;
    n.sizemask = realsize-1;
    n.table = zcalloc(realsize*sizeof(dictEntry*));
    n.used = 0;

    /* 如果以前的哈希表为空，那么就直接使用新哈希表即可，不需要进行数据移动 */
    if (d->ht[0].table == NULL) {
        d->ht[0] = n;
        return DICT_OK;
    }

    /* 令第二个哈希表使用新的哈希表 */
    d->ht[1] = n;
    /* rehashidx设置为0，标志开始rehash操作 */
    d->rehashidx = 0;
    return DICT_OK;
}
```

这个函数并没有直接调用rehash函数，而是仅仅将rehashidx赋值为0，标志着当前处于rehash状态。每当对数据库进行增删改查时，都会先判断当前是否处于rehash状态，如果是，那么就执行一次rehash，这样的好处是将rehash带来的时间空间消耗分摊给了每一个对数据库的操作。而如果采用直接将全部的数据rehash到ht[1]，那么当数据量很大时，rehash执行时间会很长，服务器很可能被阻塞。

需要注意的是为ht[1]申请的新大小而不是ht[0]，这是Redis进行rehash的策略，即将ht[0]中的每个键值对重新计算其哈希值，然后添加到ht[1]中，当ht[0]中所有数据都已经添加到ht[1]时，将ht[1]作为ht[0]使用

## Rehash操作

什么时候进行rehash操作呢，当对数据库进行增删改查时，会首先判断当前是否处在rehash状态

```c
/* 判断是否正在进行rehash操作(rehashidx != -1) */
/* 如果正在进行rehash，那么每次对数据库的增删改查都会进行一遍rehash操作
* 这样可以将rehash的执行时间分摊到每一个对数据库的操作上 */
if (dictIsRehashing(d)) _dictRehashStep(d);
```

dictIsRehashing是个宏定义，仅仅是判断rehashidx是否为-1，这就是上面为什么将rehashidx设置为0就代表开启rehash操作

```c
//dict.h
#define dictIsRehashing(d) ((d)->rehashidx != -1)
```

当Redis发现此时正处于rehash操作时，会执行rehash函数。不过Redis不是一次将ht[0]中所有的数据全部移动到ht[1]，而是仅仅移动一小部分。Redis采用N步渐进式执行rehash操作，即每次移动ht[0]N个下标的数据到ht[1]中

rehash函数由dictRehash完成，这个函数虽然长了点，但是还是蛮好理解的

其中需要注意的是

- rehashidx记录着当前rehash的进度，即从哈希表ht[0]的rehashidx位置开始移动数据到ht[1]中(每次移动完成后都会更改rehashidx)
- rehash的步骤是先获取ht[0]中rehashidx位置的哈希节点链表，将这个链表中所有哈希节点代表的键值对重新计算哈希值，添加到ht[1]中，然后从ht[0]中删除
- 每步移动一个下标下的整个链表，总共执行n步
- 如果已经将ht[0]的所有数据移动到ht[1]中，就释放ht[0]，将ht[1]作为ht[0]，然后重置ht[1]，将rehashidx设置为-1标志不再执行rehash操作

```c
//dict.c
/* 
 * 将第一个哈希表的键值对重新求哈希值后添加到第二个哈希表
 */
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    /* 没有进行rehash操作，返回 */
    if (!dictIsRehashing(d)) return 0;

    /* n步渐进式的rehash操作，每次移动rehashidx索引下的键值到新表 */
    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        /* 找到下一个存在哈希节点的索引 */
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            /* 如果遇到一定数量的空位置，就返回 */
            if (--empty_visits == 0) return 1;
        }
        /* 获取这个索引下的哈希节点链表头 */
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            unsigned int h;

            /* 保存相同索引下的下一个哈希节点 */
            nextde = de->next;
            /* Get the index in the new hash table */
            /* dictHashKey调用hasFunction，计算哈希值，与掩码与运算计算下标值 */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            /* h是rehash后的索引，将当前哈希节点保存在第二个哈希表的对应索引上 */
            de->next = d->ht[1].table[h];
            /* 记录对应索引的哈希节点表头 */
            d->ht[1].table[h] = de;
            /* 第一个哈希表的节点数减一 */
            d->ht[0].used--;
            /* 第二个哈希表的节点数加一 */
            d->ht[1].used++;
            /* 对下一个哈希节点进行处理 */
            de = nextde;
        }
        /* 将当前索引下的所有哈希节点从第一个哈希表中删掉 */
        d->ht[0].table[d->rehashidx] = NULL;
        /* 处理下一个索引 */
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    /* 如果第一个哈希表已经全部转移到第二个上，就说明rehash操作完成，将第二个作为第一个 */
    if (d->ht[0].used == 0) {
        /* 释放第一个哈希表 */
        zfree(d->ht[0].table);
        /* 第二个哈希表作为第一个哈希表 */
        d->ht[0] = d->ht[1];
        /* 重置第二个哈希表，以用于下次rehash */
        _dictReset(&d->ht[1]);
        /* rehashidx设置为-1，表示没有在进行rehash操作 */
        d->rehashidx = -1;
        /* 返回0表示rehash完成 */
        return 0;
    }

    /* More to rehash... */
    /* 返回1表示rehash没有完成，还需要继续 */
    return 1;
}
```

每次对数据库进行增删改查时，如果当前处于rehash状态，就执行一步rehash函数(n=1)

```c
//dict.c
/* 执行一步rehash */
static void _dictRehashStep(dict *d) {
    /* 执行一次rehash */
    if (d->iterators == 0) dictRehash(d,1);
}
```

另外，当处于rehash状态时，数据库的操作会有些不同

对数据库的添加操作会添加到ht[1]中而不再添加到ht[0]，因为ht[0]中的键值对迟早要添加到ht[1]中，还不如直接添加到ht[1]中。这样带来的另一个好处是ht[0]的数量不会增加，可以保证rehash早晚可以完成

对数据库的查找操作会从ht[0]和ht[1]两个哈希表中查找，因为ht[0]中的键值对已经有一部分放到ht[1]中了



此外，Redis还提供了另一种rehash策略，即每次rehash一定时间，也比较好理解

```c
//dict.c
/* 执行ms毫秒的rehash操作 */
int dictRehashMilliseconds(dict *d, int ms) {
    /* 保存开始时的时间 */
    long long start = timeInMilliseconds();
    int rehashes = 0;

    /* 每次rehash执行100步 */
    while(dictRehash(d,100)) {
        rehashes += 100;
        /* 判断时间是否到达 */
        if (timeInMilliseconds()-start > ms) break;
    }
    /* 返回rehash了多少步 */
    return rehashes;
}
```



# 字典的基本操作

字典的操作比较多，以增删改查为例

## 添加键值对

添加键值对操作是由dictAdd函数完成的，当在终端输入各种SET命令添加键值对时，最终也会调用这个函数

函数会先创建一个只有键没有值的哈希节点，然后为哈希节点设置值

```c
/* 向字典中添加键值对 */
int dictAdd(dict *d, void *key, void *val)
{
    /* 创建一个只有键没有值的哈希节点 */ 
    dictEntry *entry = dictAddRaw(d,key);

    if (!entry) return DICT_ERR;
    /* 将值添加到创建的哈希节点中 */
    dictSetVal(d, entry, val);
    return DICT_OK;
}
```

创建只有键没有值的操作由dictAddRaw函数完成，函数首先判断是否处于rehash状态，如果是，那么会执行一步rehash。同时如果处于rehash状态，那么会将键值对添加到ht[1]中而不是ht[0]

```c
//dict.c
/* 向字典中添加一个只有key的键值对，如果正在进行rehash操作，则需要先执行rehash再添加 */
dictEntry *dictAddRaw(dict *d, void *key)
{
    int index;
    dictEntry *entry;
    dictht *ht;

    /* 判断是否正在进行rehash操作(rehashidx != -1) */
    /* 如果正在进行rehash，那么每次对数据库的增删改查都会进行一遍rehash操作
     * 这样可以将rehash的执行时间分摊到每一个对数据库的操作上 */
    if (dictIsRehashing(d)) _dictRehashStep(d);

    /* 计算键key在哈希表中的索引，如果哈希表中已存在相同的key，则返回 */
    if ((index = _dictKeyIndex(d, key)) == -1)
        return NULL;

    /* 如果正在执行rehash，那么直接添加到第二个哈希表中 */
    /* 因为rehash操作是将第一个哈希表重新映射到第二个哈希表中
     * 那么就没必要再往第一个哈希表中添加，可以直接添加到第二个哈希表中
     * 这样做的目的是保证在rehash的过程中第一个哈希表的节点数量是一直减少的 */
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    /* 申请节点 */
    entry = zmalloc(sizeof(*entry));
    /* 将为key创建的哈希节点添加到哈希表的相应索引下 */
    entry->next = ht->table[index];
    ht->table[index] = entry;
    /* 节点数加一 */
    ht->used++;
    
    /* 设置节点的键为key */
    dictSetKey(d, entry, key);
    return entry;
}
```

## 删除键值对

删除键值对主要由dictGenericDelete函数完成，函数首先会判断是否处于rehash状态，如果是则执行一步rehash。然后在ht[0]和ht[1]中查找匹配的键值对，执行删除操作

```c
//dict.c
/* 从字典中删除键key对应的键值对，nofree代表是否释放键值对的内存空间 */
static int dictGenericDelete(dict *d, const void *key, int nofree)
{
    unsigned int h, idx;
    dictEntry *he, *prevHe;
    int table;

    /* 字典为空，没有数据，直接返回 */
    if (d->ht[0].size == 0) return DICT_ERR; /* d->ht[0].table is NULL */
    /* 如果正处于rehash状态，那么执行一步rehash */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    /* 计算键key的哈希值 */
    h = dictHashKey(d, key);

    /* 在ht[0]和ht[1]中寻找 */
    for (table = 0; table <= 1; table++) {
        /* 计算键key的索引下标(与掩码sizemask与运算) */
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        prevHe = NULL;
        while(he) {
            /* 依次比较索引下标下的哈希节点，找到键相同的那个哈希节点 */
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                /* Unlink the element from the list */
                /* 将哈希节点移除 */
                if (prevHe)
                    prevHe->next = he->next;
                else
                    d->ht[table].table[idx] = he->next;
                /* 如果需要释放内存，则将键值对的内存空间释放 */
                if (!nofree) {
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                }
                /* 释放哈希节点的内存 */
                zfree(he);
                /* 节点个数减一 */
                d->ht[table].used--;
                return DICT_OK;
            }
            prevHe = he;
            he = he->next;
        }
        /* 如果没有进行rehash，那么ht[1]中没有数据，不需要在ht[1]中寻找 */
        if (!dictIsRehashing(d)) break;
    }
    return DICT_ERR; /* not found */
}
```

## 修改键值对

修改键值对由dictReplace函数完成，如果键不存在，则执行常规的添加操作，如果存在，则修改值。实际调用的还是添加操作，因为添加操作当键存在是会返回错误

```c
//dict.c
/* 如果添加时对应的key已经存在，就替换 */
int dictReplace(dict *d, void *key, void *val)
{
    dictEntry *entry, auxentry;

    if (dictAdd(d, key, val) == DICT_OK)
        return 1;

    /* 到达这里表示键key在哈希表中已存在，从哈希表中找到该节点 */
    entry = dictFind(d, key);
  
    /* 保存以前的哈希节点，实际上是为了保存哈希节点值，因为需要释放值的内存 */
    auxentry = *entry;
    /* 将键key对应的节点值设置为val */
    dictSetVal(d, entry, val);
    /* 释放旧的哈希节点值 */
    dictFreeVal(d, &auxentry);
    return 0;
}
```

## 查找键值对

查找键值对由dictFind函数完成，首先根据键计算哈希值和下标，然后在对应位置查找是否有匹配的键，如果有则将哈希节点返回

```c
//dict.c
/* 查找键为key的哈希节点 */
dictEntry *dictFind(dict *d, const void *key)
{
    dictEntry *he;
    unsigned int h, idx, table;

    /* 字典为空 */
    if (d->ht[0].used + d->ht[1].used == 0) return NULL; /* dict is empty */
    /* 如果处于rehash状态，执行一步rehash */
    if (dictIsRehashing(d)) _dictRehashStep(d);
    /* 计算哈希值 */
    h = dictHashKey(d, key);
    /* 在ht[0]和ht[1]中搜索 */
    for (table = 0; table <= 1; table++) {
        /* 计算下标 */
        idx = h & d->ht[table].sizemask;
        /* 获得下标处的哈希节点链表头 */
        he = d->ht[table].table[idx];
        while(he) {
            /* 寻找键与key相同的哈希节点 */
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        /* 如果没有进行rehash，则没有必要在ht[1]中查找 */
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

# 小结

字典是Redis存储数据的数据结构，保存有两个哈希表。当字典中数据过多时会执行rehash操作，rehash操作实际是将第一个哈希表的键值对重新计算哈希值添加到第二个哈希表中。Redis没有一下子将所有键值对都进行rehash，而是采用渐进式的方法，每次对数据库进行操作时，都会执行一次rehash操作。这样就将rehash的消耗分摊到对数据库的操作上。

另外，没有找到当字典数量过少时进行的缩小操作...