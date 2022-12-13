# Redis基础

## 基础相关概念

### Redis相关数据结构

redis中经常被使用到的数据结构包含String、List、ZSet、Hash、Set

扩展数据结构包含HyperLogLogs、Bitmap、Geospatial

### Redis String相关

String无论是在redis中还是在日常开发中都是作为最常用的数据结构，java实现的String底层使用的是byte数组，而redis实现的SDS底层使用的其实也还是char数组，但是额外增加了很多内容。

#### Redis String的实现SDS

redis使用C语言进行编码，在C语言中表示字符串可以使用 char* 来表示。但是 char* 存在一些弊端。

- 获取长度时需要进行遍历，此外还需要手动检查和分配字符串空间，在操作字符串追加时比较复杂
- 使用字符”\0“进行字符串结束标识，导致正常字符串若是出现结束标识符会被截断

于是redis设计了SDS（简单动态字符串）数据结构，包含字符数组长度、字符数组分配的空间长度、字符串类型、字符数组。

```text
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* 字符数组现有长度*/
    uint8_t alloc; /* 字符数组的已分配空间，不包括结构体和\0结束字符*/
    unsigned char flags; /* SDS类型*/
    char buf[]; /*字符数组*/
};
```

所以SDS本质还是字符数组，但是额外增加了三大元数据。三大元数据带来了很多操作效率上的提升：

- 可以直接获取数据长度而不需要再去遍历字符数组来获取
- 同时由于redis采用预分配原则，在记录了分配的空间长度的同时进行字符串追加的时候若是判断在申请空间范围内，不需要再申请空间，效率提升。

此外redis对于内存的使用十分极致，仅仅是为了能够灵活的保存不同大小的字符串，避免统一规格导致小字符串结构体元数据占用多一丁点的内存，SDS就设计有四大类型（排除不适用的sdshdr5）：

- sdshdr8
  - 其len和alloc元数据的数据类型使用uint8_t，能表示不超过2的8次方也就是256，因此在此类型下字符串最大长度不超过256字节。
- sdshdr16
  - 其len和alloc元数据的数据类型使用uint16_t，能表示不超过2的16次方。
- sdshdr32
  - 其len和alloc元数据的数据类型使用uint32_t，能表示不超过2的32次方。
- sdshdr64
  - 其len和alloc元数据的数据类型使用uint64_t，能表示不超过2的64次方。

更为极致的内存空间优化redis甚至在编译sdshdr8结构时使用 ”__ attribute__ ((__ packed__))“，不进行内存对齐，也就是哪怕这个变量不足8字节，在内存对齐情况下也会给你分配8字节内存。而reids则采用紧凑型方式分配，结构体多大就分配多大内存。

### Redis Hash相关

Hash表也是常用的数据结构之一，O(1)复杂度的快速查找，和Java的Hash表类似，redis的Hash表也需要处理Hash冲突和扩容问题。

#### Redis Hash结构

Redis 中和 Hash 表实现相关的文件主要是 dict.h 和 dict.c。dict.h 文件定义了Hash表的结构、哈希项，以及Hash表的各种操作函数，而dict.c文件包含了Hash表各种操作的具体实现代码。

Hash 表被定义为一个二维数组（dictEntry **table），这个数组的每个元素是一个指向哈希项（dictEntry）的指针。

```text
typedef struct dictht {
    dictEntry **table; //二维数组
    unsigned long size; //Hash表大小
    unsigned long sizemask;
    unsigned long used;
} dictht;
```

为了实现链式哈希， Redis 在每个 dictEntry 的结构设计中，除了包含指向键和值的指针，还包含了指向下一个哈希项的指针。dictEntry 结构体中包含了指向另一个 dictEntry 结构的指针 *next。

```text
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

同时为了实现渐进式Rehash，再dict.h 文件中定义了一个dict结构体，结构体中有一个数组（ht[2]），包含了两个Hash表ht[0]和ht[1]，一般由ht[0]提供服务，rehash时，键值对迁移到ht[1]，rehash完成时 ht[1]的地址赋值给 ht[0]，ht[1]的表大小设置为 0。

```text
typedef struct dict {
    …
    dictht ht[2]; //两个Hash表，交替使用，用于rehash操作
    long rehashidx; //Hash表是否在进行rehash的标识，-1表示没有进行rehash，同时这也是标志着渐进式rehash需要执行的bucket位置
    …
} dict;
```

**Hash表中节省内存的操作**
在 dictEntry 结构体中，键值对的值是由一个联合体 v 定义的，v 中包含了指向实际值的指针 *val，还包含了无符号的 64 位整数、有符号的 64 位整数，以及 double 类的值。这种实现方法是一种节省内存的开发小技巧。当值为整数或双精度浮点数时，由于其本身就是 64 位，就可以不用指针指向了，而是可以直接存在键值对的结构体中，这样就避免了再用一个指针，从而节省了内存空间。

#### Redis Hash冲突解决方案

所谓Hash冲突就是不同的key定位到同一个Hash桶上，一般而言Hash冲突有两种解决方式，一种是拉链法，一种是开放地址法。

开放地址法：当key定位到Hash表同一个桶位置时则顺着表找到空桶给放进去，在java中ThreadLocalMap就是使用开发地址法解决冲突。

拉链法：使用链式结构存储，挨个放到链表里去。java中的HashMap和redis中的hash表都是使用这种方式，但是java的HashMap还有链表转红黑树的优化。

#### Redis Hash表扩容rehash

Hash表扩容的代码_dictExpandIfNeeded：

```text
if (d->ht[0].size == 0) 
   return dictExpand(d, DICT_HT_INITIAL_SIZE);
 
if (d->ht[0].used >= d->ht[0].size &&(dict_can_resize ||
              d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
{
    return dictExpand(d, d->ht[0].used*2);
}
```

Hash表扩容的条件：

- ht[0]的大小为0，其实主要是初始化操作，不属于rehash。
- ht[0]承载的元素个数已经超过了 ht[0]的大小 并且 （Hash 表可以进行扩容 或者 承载元素个数是ht[0]大小的五倍以上）。
  - 这里是前后两个条件，AND之前的条件很好理解，主要看后面的。
    - Hash是否可以扩容主要看dict_can_resize变量值，这个变量值是在 dictEnableResize 和 dictDisableResize 两个函数中设置的
      - 而这两个函数被封装在updateDictResizePolicy函数中来启动或禁用rehash扩容。
      - 简要一句话就是，在当前没有 RDB 子进程，并且也没有 AOF 子进程。dict_can_resize就会被设置为1表明可以扩容，否则就会被设置为0，不允许扩容。
    - 至于ht[0]承载元素个数是ht[0]大小的五倍以上也好理解。
- 综合来说，是否可以rehash首先看承载的元素是否大于ht[0]的大小，大于但是不到五倍，如果此时没有RDB子进程也没有AOF子进程那就扩容，如果超过五倍那就是无条件扩容。

**Redis什么时候会触发扩容判断？**

这里就看那里调用了_dictExpandIfNeeded，顺序为dictAddRaw函数调用_dictKeyIndex函数，而_dictKeyIndex函数就调用了dictExpandIfNeeded。dictAddRaw则被以下三个函数调用：

- dictAdd
  - 用来往 Hash 表中添加一个键值对。
- dictRelace
  - 用来往 Hash 表中添加一个键值对，或者键值对存在时，修改键值对。
- dictAddorFind
  - 直接调用 dictAddRaw。

综上，往 Redis 中写入新的键值对或是修改键值对时，Redis 都会判断下是否需要进行 rehash。

**Redis每次扩容会扩容多大？**

Hash表空间的扩容主要通过调用dictExpand函数来完成

```text
int dictExpand(dict *d, unsigned long size);
```

而在_dictExpandIfNeeded代码中我们可以看到参数值为，hash表当前已使用值的两倍。

```text
if (d->ht[0].size == 0) 
   return dictExpand(d, DICT_HT_INITIAL_SIZE);
 
if (d->ht[0].used >= d->ht[0].size &&(dict_can_resize ||
              d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
{
    return dictExpand(d, d->ht[0].used*2);
}
```

但是在dictExpand函数中，具体执行是由_dictNextPower函数完成

```text
static unsigned long _dictNextPower(unsigned long size)
{
    //哈希表的初始大小
    unsigned long i = DICT_HT_INITIAL_SIZE;
    //如果要扩容的大小已经超过最大值，则返回最大值加1
    if (size >= LONG_MAX) return LONG_MAX + 1LU;
    //扩容大小没有超过最大值
    while(1) {
        //如果扩容大小大于等于最大值，就返回截至当前扩到的大小
        if (i >= size)
            return i;
        //每一步扩容都在现有大小基础上乘以2
        i *= 2;
    }
}
```

所以其实传递进入的size值虽然是Hash表当前使用量的两倍，但是内部实际会采用DICT_HT_INITIAL_SIZE值逐次乘以2之后刚好大于size值的数为扩容之后的大小，也就是大于当前Hash表已使用容量两倍最小的2的n次幂。举个例子就是，假如当前Hash表已使用的大小为30，那么实际扩容后的结果是64，而不是60。

#### Redis 渐进式rehash

一般而言，如果一个hash表需要触发扩容，那么会一次性扩容并拷贝迁移元素完成，但是在拷贝时，由于Redis主线程无法执行其他请求，所以会阻塞主线程，这样就会产生rehash开销。于是Redis就提出了渐进式rehash的方法。渐进式rehash的意思就是Redis并不会一次性把当前Hash表中的所有键都拷贝到新位置，而是会分批拷贝，每次的键拷贝只拷贝Hash表中一个bucket中的哈希项。这样每次键拷贝的时长有限，对主线程的影响也就有限了。

**渐进式rehash的实现方式**

rehash底层由函数dictRehash来进行实现：

```text
nt dictRehash(dict *d, int n) {
    int empty_visits = n*10;
    ...
    //主循环，根据要拷贝的bucket数量n，循环n次后停止或ht[0]中的数据迁移完停止
    while(n-- && d->ht[0].used != 0) {
        //如果当前要迁移的bucket中没有元素
        while(d->ht[0].table[d->rehashidx] == NULL) {
            //
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        //如果rehashidx指向的bucket不为空
        while(de) {
            uint64_t h;
            //获得同一个bucket中下一个哈希项
            nextde = de->next;
            //根据扩容后的哈希表ht[1]大小，计算当前哈希项在扩容后哈希表中的bucket位置
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            //将当前哈希项添加到扩容后的哈希表ht[1]中
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            //减少当前哈希表的哈希项个数
            d->ht[0].used--;
            //增加扩容后哈希表的哈希项个数
            d->ht[1].used++;
            //指向下一个哈希项
            de = nextde;
        }
        //如果当前bucket中已经没有哈希项了，将该bucket置为NULL
        d->ht[0].table[d->rehashidx] = NULL;
        //将rehash加1，下一次将迁移下一个bucket中的元素
        d->rehashidx++;
    }
    //判断ht[0]的数据是否迁移完成
    if (d->ht[0].used == 0) {
        //ht[0]迁移完后，释放ht[0]内存空间
        zfree(d->ht[0].table);
        //让ht[0]指向ht[1]，以便接受正常的请求
        d->ht[0] = d->ht[1];
        //重置ht[1]的大小为0
        _dictReset(&d->ht[1]);
        //设置全局哈希表的rehashidx标识为-1，表示rehash结束
        d->rehashidx = -1;
        //返回0，表示ht[0]中所有元素都迁移完
        return 0;
    }
    //返回1，表示ht[0]中仍然有元素没有迁移完
    return 1;
}
```

所以dictRehash函数整体逻辑为

- 首先会根据传入的n执行一个循环，n就代表了需要拷贝的bucket数量，依次完成这些bucket内部所有键的迁移。当然，如果ht[0]哈希表中的数据已经都迁移完成了，键拷贝的循环也会停止执行。
- 完成了n个bucket拷贝后，dictRehash函数会判断 ht[0]表中数据是否都已迁移完。如果都迁移完了，那么ht[0]的空间会被释放。因为 Redis在正常处理请求时，代码逻辑中都是使ht[0]，所以当rehash执行完成后，虽然数据都在ht[1]中了，但Redis仍然会把ht[1]赋值给ht[0]，以便其他部分的代码逻辑正常使用。
- 在 ht[1]赋值给ht[0]后，它的大小就会被重置为0，等待下一次rehash。与此同时，全局哈希表中的rehashidx变量会被标为-1，表示rehash结束了

在dict结构中讲过rehashidx用来标识是否在进行rehash，其原理就是rehashidx表示的是当前rehash在对哪个bucket做数据迁移，-1就是没有在rehash，0就标识对 ht[0]中的第一个bucket进行数据迁移，所以同样的由于在rehash过程中数据分布在ht[0]和ht[1]上，通过rehashidx也可以知道对应key的数据存在哪个上面从而去获取数据。

**渐进式rehash中遇到空bucket处理**

在dictRehash函数的主循环中会判断rehashidx指向的bucket是否为空，如果为空，那就将rehashidx的值加1，检查下一个bucket。在遇到多个连续bucket为空时，渐进式 rehash 不会一直递增rehashidx进行检查，代码中可以看到empty_visits进行了次数限制，避免连续空bucket过多导致Redis主线程长期无法处理其他请求。

而empty_visits设置的值为传入的n值乘以10，对于外部调用dictRehash函数传入的n值都为1，表示每次只处理一个bucket的拷贝，所以遇到连续空bucket的情况下，最多检查10次，连续10空bucket也会停止这次rehash。

**什么时候会触发渐进式rehash以及为何每次只处理1个bucket？**

这里主要就是看什么地方调用了dictRehash函数，_dictRehashStep实现了调用dictRehash函数，调用时固定了传入的n值为1。

```text
static void _dictRehashStep(dict *d) {
    //给dictRehash传入的循环次数参数n为1
    if (d->iterators == 0) dictRehash(d,1);
}
```

而什么时候触发渐进式rehash，则要看_dictRehashStep函数在哪里被调用，这里共有五个函数：dictAddRaw，dictGenericDelete，dictFind，dictGetRandomKey，dictGetSomeKeys。对了增、删、查操作都会触发一次渐进式rehash。
























**参考**

1.《Redis源码剖析与实战》
