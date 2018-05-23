---
title : Redis源码剖析（十三）整数集合
date : 2018-01-20 12:36
categories : Redis
tags : Redis
---



Redis提供一种叫整数集合的数据结构，当数据中只包含整数，并且数据数量不多时，Redis便会采用整数集合存储

Redis保证整数集合有以下几个特性

* 所含元素全是整数，且不重复
* 内部元素有序，通常是会从小到大排序
* 内部编码统一，尽可能采用合适的编码保存数据
* 当编码不合适时，执行升级操作

<!--more-->

接下来会针对上述几个特性分别进行分析，可以看到，整数集合有点类似连续数组，只是在某种程度上添加了编码，同时为了编码统一，会有升级相关操作

# 命令格式

Redis提供SADD命令向数据库中添加整数集合

```c
127.0.0.1:6379> SADD digits 1 2 3 4 5	//向数据库中添加整数集合
(integer) 5
127.0.0.1:6379> SMEMBERS digits	//获取整数集合digits的成员
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
127.0.0.1:6379> OBJECT ENCODING digits	//获取digits内部存储结构
"intset"
127.0.0.1:6379> 
```

不过，有以下几种情况Redis会更改底层的存储结构，将整数集合改为哈希表

- 元素个数过多时
- 存在非整数元素时

当采用SADD向整数集合中添加元素，但是其中包含一个字符串类型数据时，那么再次获取内部存储结构时会发现返回的是"hashtable"而非"intset"

```c
127.0.0.1:6379> SADD digits-str 1 2 3 a 4 5	//其中带有字符串a
(integer) 6
127.0.0.1:6379> SMEMBERS digits-str	//获取元素
1) "1"
2) "3"
3) "2"
4) "4"
5) "a"
6) "5"
127.0.0.1:6379> OBJECT ENCODING digits-str	//获取内部存储结构，发现是哈希表
"hashtable"
127.0.0.1:6379> 
```

# 存储结构

前面也提高过，整数集合很像连续数组，内部保存的是整数。但是Redis为整数集合添加了编码的功能，也就是类型。可以根据元素大小，选择合适的编码来存储，当然，是为了节约内存

Redis提供三种编码，分别对应int16\_t，int32\_t，int64_t三种类型，对于采用int16\_t就可以存储的数据，采用int32\_t或int64\_t就显得过于浪费了，这便是编码的实际作用，采用最适合的类型保存数据

```c
//intset.c
#define INTSET_ENC_INT16 (sizeof(int16_t))
#define INTSET_ENC_INT32 (sizeof(int32_t))
#define INTSET_ENC_INT64 (sizeof(int64_t))
```

整数集合的定义如下，其中保存着内部元素的编码，元素个数，和元素数组

```c
//intset.h
/* 整数集合，用于节约内存 */
typedef struct intset {
    uint32_t encoding; //编码，不同的整数大小(2, 4, 8字节)
    uint32_t length; //多少个数据
    int8_t contents[]; //数据
} intset;
```

需要注意的是，整数集合中的所有数据的编码都是一样的，也就是说，如果其中一个元素的编码要改变，所有元素的编码都需要同时改变，这一点在后面添加元素时会看到

另外，虽然contents数组中的元素类型是int8_t类型，但是这并不代表数据是这个类型的。int8\_t只是为了用最小的类型记录数据，在存放数据时，一个数据可以同时占用多个int8\_t(这是由于内部数据的地址空间是连续的)

# 整数集合相关操作

## 添加数据

添加数据功能由intsetAdd函数完成，函数内部首先判断要添加的数据是否能被当前编码保存，如果不能，则需要将整个集合的数据重新改写编码，也就是升级操作

```c
//intset.c
/* 在整数集中添加一个元素 */
/* 根据编码长度判断是否需要先执行升级操作再添加 */
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;

    /* 如果要添加的数据无法被当前编码保存，就需要升级操作 */
    if (valenc > intrev32ifbe(is->encoding)) {
        return intsetUpgradeAndAdd(is,value);
    } else {
        /* 如果value已在集合中，则不添加 */
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        /* 扩充集合，为新数据分配空间 */
        is = intsetResize(is,intrev32ifbe(is->length)+1);
        /* 为了保证集合中元素有序，需要执行移动操作 */
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    /* 将新数据放在pos位置 */
    _intsetSet(is,pos,value);
    /* 更新集合数据个数 */
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

## 移动数据

intsetMoveTail函数用于将源下标开始的数据移动到目的下标处

```c
//intset.c
/* 将整数集合中从from开始的数据移动到to位置 */
static void intsetMoveTail(intset *is, uint32_t from, uint32_t to) {
    void *src, *dst;
    /* 计算要移动的字节数 */
    uint32_t bytes = intrev32ifbe(is->length)-from;
    uint32_t encoding = intrev32ifbe(is->encoding);

    if (encoding == INTSET_ENC_INT64) {
        src = (int64_t*)is->contents+from;
        dst = (int64_t*)is->contents+to;
        bytes *= sizeof(int64_t);
    } else if (encoding == INTSET_ENC_INT32) {
        src = (int32_t*)is->contents+from;
        dst = (int32_t*)is->contents+to;
        bytes *= sizeof(int32_t);
    } else {
        src = (int16_t*)is->contents+from;
        dst = (int16_t*)is->contents+to;
        bytes *= sizeof(int16_t);
    }
    /* memmove，内存移动操作 */
    memmove(dst,src,bytes);
}
```

## 升级操作

如果不考虑升级操作，添加函数还是比较容易理解的。找到新数据的位置，然后移动元素，将新元素放到它的位置上，这些操作和数组的添加操作非常像。

前面说过，编码的加入是为了节约内存占用，但是带来的问题就是内部编码统一，整个集合都需要采用相同的编码保存数据，那么当一个数据无法被当前编码保存时，就需要将整个集合的编码升级，这就导致所有原有数据的编码也要被改变

举例来说，假设之前采用int16_t就可以保存所有数据，此时需要一个int32\_t类型才能保存的数据，那么就需要将以前的数据都改为int32\_t类型以保证编码统一

>编码统一的原因是整数集合内部采用数组保存数据，每个数据的大小都必须是一样的，这样才可以通过偏移量(下标)来获取数据

intsetUpgradeAndAdd函数先将集合编码升级，然后再添加数据

```c
//intset.c
/* 将数据插入到整数集中，如果当前整数集的编码不足以容纳value，那么将整数集执行升级操作 */
/* 升级操作是将整数集的编码加大，这需要将原有数据的编码也进行加大 */
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    /* 当前整数集的编码 */
    uint8_t curenc = intrev32ifbe(is->encoding);
    /* 适合value的最小编码 */
    /* 其实就是根据value的大小找到一个编码使其不溢出 */
    uint8_t newenc = _intsetValueEncoding(value);
    /* 整数集中数据个数 */
    int length = intrev32ifbe(is->length);
    /* value为正，则在尾部插入，否则在头部插入 */
    int prepend = value < 0 ? 1 : 0;

    /* 将整数集的编码设置成新编码 */
    is->encoding = intrev32ifbe(newenc);
    /* 多申请一个空间存放value，此时申请的空间是根据新编码大小进行分配的 */
    is = intsetResize(is,intrev32ifbe(is->length)+1);
    /* 将原有数据进行调整，加大其编码长度，同时改变在整数集的位置以便容纳新元素 */
    /* _intsetGetEncoded函数根据给定编码获取整数集中的某个下标元素(此处通过源编码找到以前的元素)
     * _intsetSet函数将给定元素添加到整数集的某个下标位置，根据当前编码 */
    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));

    /* 根据在头部插入还是在尾部插入将value插进整数集中 */
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    /* 更新元素个数 */
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

>Redis的整数集合不支持降级操作，也就是一旦将编码调高，就无法将其降低，这是没有办法的事情，因为如果要降级，就需要遍历数据判断是否需要降级，这个操作是十分耗时的

# 对象系统中的整数集合

整数集合在对象系统中作为集合的底层实现

```c
//object.c
/* 创建整数集合对象 */
robj *createIntsetObject(void) {
    intset *is = intsetNew();
    robj *o = createObject(OBJ_SET,is);
    o->encoding = OBJ_ENCODING_INTSET;
    return o;
}
```

# 小结

整数集合部分还是很容易理解的，实际上就是数组外套一个编码，根据编码统一适当进行升级操作。另外，整数集合作为集合的底层实现，保证了数据的有序性，无重复性，但是只适用于数据个数较少，且都是整数的情况，当数据个数很多，或者存在其他类型的数据(如字符串)时，Redis会采用hashtable作为集合的底层实现