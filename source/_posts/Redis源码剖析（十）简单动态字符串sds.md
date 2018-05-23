---
title : Redis源码剖析（十）简单动态字符串sds
date : 2018-01-10 16:33
categories: Redis
tags : Redis
---



在对象系统概述中发现，好像所有和字符串有关的内容都有sds的存在，实际上，它是Redis内部对于c字符串的封装，所谓c字符串，其实就是char *，在sds.h头文件中可以清楚的看到它的定义

```c
//sds.h
typedef char *sds;
```

所以创建sds类型变量实际上就是创建了一个char*变量，不过Redis字符串独特的地方在于它记录了字符串的长度。考虑想要计算c原生字符串的长度，需要使用strlen函数，该函数会遍历字符串直到遇到'\0'，复杂度是O(n)，这容易造成性能瓶颈，所以Redis在创建字符串时记录了字符串的长度，只需要O(1)即可计算得到

<!--more-->

# 字符串结构

Redis字符串不仅仅保存实际的数据，还保存着用于记录长度的头部信息。假设要创建一个长度为n的字符串，那么Redis会申请大于n的内存空间，其中，后半部分存储字符串数据，前半部分保存字符串头部信息，Redis会根据字符串的长度选择不同的头部编码

```c
//sds.h
/* 
 * 字符串Header的定义，在字符串前面存放一个Header，用于记录字符串的信息
 * __attribute__ ((__packed__))是通知编译器使用紧凑模式分配内存
 * 由于内存对齐的原因，编译器可能会在任意位置添加字节以满足对齐条件
 * 该声明告知编译器在成员变量之间不能插入字节，要插就在头尾插
 * 原因是如果在中间插会破坏内存结构，导致直接进行char*加法无法准确获取数据 
 * */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; 
    char buf[];
};

/* 
 * 不同长度字符串采用不同的Header结构以节约内存
 * len : 字符串数据真正长度，已容纳的字符数，不包括结尾字符'\0'
 * alloc : 字符串数据的最大容量，可以容纳多少个字符，不包括结尾字符'\0'
 * flags : Header的类型，总共5中，但是sdshdr5已经不再使用
 * buf : 实际存储字符串的位置
 */
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; 
    uint8_t alloc; 
    unsigned char flags; 
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; 
    uint16_t alloc; 
    unsigned char flags; 
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; 
    uint32_t alloc; 
    unsigned char flags; 
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; 
    uint64_t alloc; 
    unsigned char flags;
    char buf[];
};
```

其中，flags记录着当前头部采用的编码格式，由宏定义指出

```c
//sds.h
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
```

这么做的原因是为了节省内存，如果一个字符串的长度使用uint8_t就可以保存，那么使用uint64\_t就显得多余了，还记得吗，Redis正在努力节省内存

前面也看到了，sds就是char*的类型别名，所以平常使用的sds实际上是指sdshdr中的buf部分，那么怎么获取头部信息呢，Redis保证为sdshdr和buf中的字符串申请的内存是连续的，也就是说如果sds指向buf的首地址，那么sds-1就指向头部信息中的flags地址，sds-n就可以获取长度和容量地址(n需要根据flags表示的编码格式计算)

由于Redis中使用长度记录字符串的结束位置，而不是依赖于'\0'，这使Redis中的字符串是二进制安全的，因为在二进制文件中，可能会存在多个'\0'，如果仅仅使用c语言原生字符串，那么很多数据都会丢失

# 字符串操作

## 创建字符串

字符串创建工作由sdsnewlen函数完成，函数首先根据字符串长度找到合适的头部编码，然后一次性申请头部和字符串数据的内存，完成初始化工作后，返回字符串数据

```c
//sds.c
/* 申请长度为initLen的字符串，如果init不为空，那么将init的数据复制到字符串中 */
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    /* 根据字符串长度选择合适的头部编码 */
    char type = sdsReqType(initlen);
    /* sdshdr5已经不再使用，默认使用sdshdr8代替 */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    /* 获取选择的头部大小，用于申请内存 */
    int hdrlen = sdsHdrSize(type);
    unsigned char *fp; /* flags pointer. */

    /* 宏定义，调用的是zmalloc, 加1是为了保存'\0' */
    /* 一次性申请可以保存头部信息和字符串数据的内存空间，确保内存是连续的 */
    sh = s_malloc(hdrlen+initlen+1);
    /* 如果给定的地址是null，那么就初始化分配的内存为0，否则，执行拷贝工作 */
    if (!init)
        memset(sh, 0, hdrlen+initlen+1);
    if (sh == NULL) return NULL;
    /* sh指向头部，为了获取字符串数据，需要跳过头部内存 
     * s指向实际用于保存数据的地址 */
    s = (char*)sh+hdrlen;
    /* s-1是Header中flags的地址 */
    fp = ((unsigned char*)s)-1;
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            /* 获取不同类型的Header的指针 */
            SDS_HDR_VAR(8,s);
            /* 设置属性，首次申请的容量和数据长度相等 */
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
    }
    /* 如果给定地址不为null，执行拷贝工作 */
    if (initlen && init)
        memcpy(s, init, initlen);
    /* 设置结束字符，多申请1个字节的原因 */
    s[initlen] = '\0';
    /* 返回字符串数据部分 */
    return s;
}
```

SDS_HDR_VAR是宏定义，根据头部编码和字符串数据地址s获取头部地址，只需要数据地址减去头部长度即可，其中##用于连接左右两部分

```c
//sds.h
/* 获取字符串Header的属性值，##作用是将前后两部分连在一起，即sdshdr##32 == sdshdr32(此时T为32) */
#define SDS_HDR_VAR(T,s) struct sdshdr##T *sh = (void*)((s)-(sizeof(struct sdshdr##T)));
```

sdsReqType函数用于根据字符串长度选择合适的头部编码

```c
//sds.c
/* 根据字符串长度选择合适的头部编码，需要找到足够容纳并且最小的编码格式 */
static inline char sdsReqType(size_t string_size) {
    /* 2的5次方，使用SDS_TYPE_5 */
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    /* 2的8次方，使用uint8_t */
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    /* 使用uint16_t */
    if (string_size < 1<<16)
        return SDS_TYPE_16;
    /* uint32_t */
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
    /* uint64_t */
    return SDS_TYPE_64;
}
```



## 获取字符串长度

获取字符串长度只需要从头部信息中读取，时间复杂度为O(1)

```c
//sds.h
/* 获取字符串长度，先获取头部编码，然后地址前移，找到头部地址，获取长度 */
static inline size_t sdslen(const sds s) {
    /* 获取头部编码 */
    unsigned char flags = s[-1];
    /* 选择对应的头部编码，获取头部地址，返回长度 */
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return SDS_TYPE_5_LEN(flags);
        case SDS_TYPE_8:
            return SDS_HDR(8,s)->len;
        case SDS_TYPE_16:
            return SDS_HDR(16,s)->len;
        case SDS_TYPE_32:
            return SDS_HDR(32,s)->len;
        case SDS_TYPE_64:
            return SDS_HDR(64,s)->len;
    }
    return 0;
}
```

>因为头部编码类型的宏定义分别是0,1,2,3,4，所以这里flags与类型掩码做与运算只是让flags的值在0到4之间，SDS_TYPE_MASK实际上是7

## 获取字符串剩余容量

获取字符串的剩余容量只需要用总容量减去已用容量，由sdsavail函数完成，函数大体和sdslen相同

```c
//sds.h
/* 
 * 返回剩余可利用的内存大小，利用Header中的alloc和len属性
 * alloc : 保存字符串的总容量
 * len : 实际存储的大小
 */
static inline size_t sdsavail(const sds s) {
    unsigned char flags = s[-1];
    switch(flags&SDS_TYPE_MASK) {
        case SDS_TYPE_5: {
            return 0;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            //和sdslen函数相比仅仅是返回数据不同
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            return sh->alloc - sh->len;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            return sh->alloc - sh->len;
        }
    }
    return 0;
}
```

## 字符串容量扩充

sdsMakeRoomFor函数用于将字符串扩充，扩充的目的通常用于和其他字符串拼接

```c
//sds.c
/* 为s的字符串申请更大的容量已容纳addLen个字节(Header中的alloc记录当前字符串的总容量) */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    void *sh, *newsh;
    /* 获取s字符串剩余可利用的字节大小,Header中的总容量(alloc) - 当前大小(len) */
    size_t avail = sdsavail(s);
    size_t len, newlen;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;

    /* Return ASAP if there is enough space left. */
    /* 如果可用空间足够，直接返回，不需要扩充 */
    if (avail >= addlen) return s;

    /* 返回s字符串的已用大小(Header中的len) */
    len = sdslen(s);
    /* 获取字符串s的头部地址 */
    sh = (char*)s-sdsHdrSize(oldtype);
    /* 新的容量等于当前已用空间加扩充大小 */
    newlen = (len+addlen);
    /* 每次扩充都会申请多余空间以备不时之需 */
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    /* 为新容量找到一个合适的头部编码 */
    type = sdsReqType(newlen);

    if (type == SDS_TYPE_5) type = SDS_TYPE_8;

    /* 计算新编码的Header大小 */
    hdrlen = sdsHdrSize(type);
    /* 如果新编码和之前编码相同，只需要扩充数据即可
     * 否则，重新申请内存(因为不同编码的Header大小不同，同时也需要扩充头部) */
    if (oldtype==type) {
        /* 在原地址处重新申请内存 */
        newsh = s_realloc(sh, hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        /* 获取字符串数据地址 */
        s = (char*)newsh+hdrlen;
    } else {
        /* 重新申请新内存 */
        newsh = s_malloc(hdrlen+newlen+1);
        if (newsh == NULL) return NULL;
        /* 将原数据拷贝到新内存中 */
        memcpy((char*)newsh+hdrlen, s, len+1);
        /* 释放原内存 */
        s_free(sh);
        /* 获取数据地址 */
        s = (char*)newsh+hdrlen;
        /* 设置Header编码 */
        s[-1] = type;
        /* 设置字符串长度 */
        sdssetlen(s, len);
    }
    /* 更新字符串容量 */
    sdssetalloc(s, newlen);
    return s;
}
```

## 连接两个字符串

前面也说到了，扩充容量通常是为了拼接其他字符串，由sdscatlen实现

```c
//sds.c
/* 连接两个字符串 */
sds sdscatlen(sds s, const void *t, size_t len) {
    size_t curlen = sdslen(s);

    /* 为s扩充容量，足以容纳字符串t(长度为len) */
    s = sdsMakeRoomFor(s,len);
    if (s == NULL) return NULL;
    /* 将字符串复制到s的尾部 */
    memcpy(s+curlen, t, len);
    /* 更新字符串已用容量 */
    sdssetlen(s, curlen+len);
    /* 设置结束字符 */
    s[curlen+len] = '\0';
    return s;
}
```

## 字符串容量缩减

扩充和缩减是相反的两个过程，代码也非常相似，扩充是令容量增加，缩减是令容量减小

```c
//sds.c
/* 将字符串末尾的空闲空间释放掉，即另alloc == len */
sds sdsRemoveFreeSpace(sds s) {
    void *sh, *newsh;
    char type, oldtype = s[-1] & SDS_TYPE_MASK;
    int hdrlen;
    /* 获取当前字符串的长度len */
    size_t len = sdslen(s);
    sh = (char*)s-sdsHdrSize(oldtype);

    /* 找到合适的Header编码(有可能编码变小) */
    type = sdsReqType(len);
    hdrlen = sdsHdrSize(type);
    /* 判断Header的类型和原先是否相同，如果不相同，需要重新分配内存
     * 因为Hedaer大小会改变 */
    if (oldtype==type) {
        newsh = s_realloc(sh, hdrlen+len+1);
        if (newsh == NULL) return NULL;
        s = (char*)newsh+hdrlen;
    } else {
        newsh = s_malloc(hdrlen+len+1);
        if (newsh == NULL) return NULL;
        memcpy((char*)newsh+hdrlen, s, len+1);
        s_free(sh);
        s = (char*)newsh+hdrlen;
        s[-1] = type;
        sdssetlen(s, len);
    }
    sdssetalloc(s, len);
    return s;
}
```

# 对象系统中的sds

在之前对象系统的介绍中，发现很多地方用到了sds变量，现在就来回忆一下，顺便填填坑

## 创建Raw类型和编码的字符串对象

创建raw字符串由createRawStringObject函数完成，函数中调用sdsnewlen创建了一个sds字符串

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

    /* 计算当前时间，赋值给lru作为最后一次访问时间 */
    o->lru = LRU_CLOCK();
    /* 返回对象指针 */
    return o;
}

/* 创建raw字符串类型变量 */
robj *createRawStringObject(const char *ptr, size_t len) {
    /* sdsnewlen()创建一个长度为len的sds字符串 */
    return createObject(OBJ_STRING,sdsnewlen(ptr,len));
}
```

可以看到，raw字符串调用了两次动态内存申请，一次在sdsnewlen函数中申请头部和字符串数据内存，一次在createObject函数中申请robj内存

## 创建Embstr类型和编码的字符串对象

创建embstr字符串由createEmbeddedStringObject函数完成

```c
//object.c
/* 创建类型为embstr，编码为embstr的字符串对象 */
robj *createEmbeddedStringObject(const char *ptr, size_t len) {
    /* 一次性创建robj和sds内存
     * 因为embstr仅仅用于长度较小的字符串，所以使用sdshdr8就足够了*/
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
    /* o是robj*类型，o+1实际加的字节数是sizeof(robj)，得到的是sds头部地址 */
    struct sdshdr8 *sh = (void*)(o+1);

    /* 设置类型，编码，数据，引用计数，最后一次访问时间 */
    o->type = OBJ_STRING;
    o->encoding = OBJ_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    o->lru = LRU_CLOCK();

    /* 设置sds头部信息，复制数据到sds中 */
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

可以看到，embstr字符串只调用了一次动态内存申请，一次性申请了robj，sds头部和字符串数据所需要的所有内存，所以embstr通常用于长度较小的字符串编码，因为动态内存申请是很耗时的，尤其是申请大内存时

# 小结

sds实际上就是char*类型，Redis在c原生字符串的基础上添加了头部信息，用于以O(1)的时间复杂度获取字符串长度。字符串操作的实现都比较简单，其中由于头部信息和数据是连续存储的，所以可以使用类似s[-n]的形式索引到头部信息