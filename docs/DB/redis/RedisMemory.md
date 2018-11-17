# 理解Redis内存
Redis的所有的数据都是存在了内存中的，虽然现在内存越来越便宜，但是跟平时电脑上装的硬盘相比，硬盘的价格就是个渣渣。内存还是非常宝贵的,就拿我的一台腾讯云的服务器来说，目前是1核2G的，但是要想升级到4G，就得需要多掏1000大洋。这些钱感觉我都可以买个1T的硬盘了。。。这就是差距。so，如何合理高效的利用Redis内存就变得非常的重要了。首先我们应该知道Redis的内存主要消耗在什么地方，其次是如何管理内存，最后才是怎么做Redis的内存优化。这样才能用更少的内存，存储更多的数据，降低成本。

## 内存消耗

如何查看Redis中内存的消耗情况哪？可以通过 `info`命令，查看Redis内存消耗的相关指标，从而有助于更好的分析内存。执行命令之后有这么几个重要的指标：
|属性名|属性说明|
| 📊 |⚔️ | 🖥 | 🚏 | 🏖  | 🌁| 📮 | 🔍 | 🚀 | 🌈 |💡
| :--------: | :---------: | :---------: | :---------: | :---------: | :---------:| :---------: | :-------: | :-------:| :------:|:------:|
| [集合](#常用集合) | [多线程](#java-多线程)|[JVM](#jvm) | [分布式](#分布式相关) |[框架](#常用框架第三方组件)|[架构设计](#架构设计)| [数据库](#db-相关) |[算法](#数据结构与算法)|[Netty](#netty-相关)| [附加技能](#附加技能)|[联系作者](#联系作者) |


| :--------: | :--------: |
|`used_memory`|Redis分配器分配的内存总量，指Redis存储的所有数据所占的内存|
|`used_memory_human`|以可读的形式返回`user_memory`|
|`used_memory_rss`|Redis进程占用的物理内存总量|
|`used_memory_peak`|`used_memory`使用的峰值|
|`used_memory_peak_human`|可读格式返回`used_memory_peak`|
|`used_memory_lua`|Lua引擎消耗的内存大小|
|`mem_fragmentation_ratio`|`used_memory_rss`/`used_memory`比值，内存碎片率|
|`mem_allocator`|Redis所使用的内存分配器，默认jemalloc|

```
used_memory:1038744104
used_memory_human:990.62M
used_memory_rss:5811122176
used_memory_peak:4563077088
used_memory_peak_human:4.25G
used_memory_lua:35840
mem_fragmentation_ratio:5.59
mem_allocator:libc
```
重点需要关注下`mem_fragmentation_ratio`这个值：
> `mem_fragmentation_ratio > 1` 说明多出来的部分名没有用于数据存储，而是被内存碎片所消耗，相差越大，说明内存碎片率越严重。
> `mem_fragmentation_ratio < 1`  一般出现在Redis内存交换（Swap）到硬盘导致（`used_memory > 可用最大内存`时，Redis会把旧的和不适用的数据写入到硬盘，这块空间就叫Swap空间），出现这种情况需要格外关注，硬盘速度远远慢于内存，Redis性能就会变得很差，甚至僵死。

### 内存消耗的划分
Redis的内存主要包括：对象内存+缓冲内存+自身内存+内存碎片。
![这里写图片描述](https://img-blog.csdn.net/20180823213024994?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)


###### 1、对象内存
对象内存是Redis内存中占用最大一块，存储着所有的用户的数据。Redis所有的数据都采用的是key-value型数据类型，每次创建键值对的时候，都要创建两个对象，key对象和value对象。key对象都是字符串，value对象的存储方式，五种数据类型--String，List，Hash，Set，Zset。每种存储方式在使用的时候长度、数据类型不同，则占用的内存就不同。
###### 2、缓冲内存
主要包括：客户端缓冲、复制积压缓冲区、AOF缓冲区
客户端缓冲：普通的客户端的连接（大量连接），从客户端（主要是复制的时候，异地跨机房，或者主节点下有多个从节点），订阅客户端（发布订阅功能，生产大于消费就会造成积压）
复制积压缓冲：2.8版本之后提供的可重用的固定大小缓冲区用于实现部分复制功能，默认1MB，主要是在主从同步时用到。
AOF缓冲区：持久化用的，会先写入到缓冲区，然后根据响应的策略向磁盘进行同步，消耗的内存取决于写入的命令量和重写时间，通常很小。

###### 3、内存碎片
目前可选的分配器有jemalloc、glibc、tcmalloc默认jemalloc
出现高内存碎片问题的情况：大量的更新操作，比如append、setrange；大量的过期键删除，释放的空间无法得到有效利用
解决办法：数据对齐，安全重启（高可用/主从切换）。

###### 4、自身内存
主要指AOF/RDB重写时Redis创建的子进程内存的消耗，Linux具有写时复制技术（copy-on-write），父子进程会共享相同的物理内存页，当父进程写请求时会对需要修改的页复制出一份副本来完成写操作。

## 管理内存

**设置上限**

Redis默认是无限使用内存。所以在使用的时候尽量的去配置`maxmemory`，给Redis设置内存使用上限，防止因Redis的无限使用造成系统内存耗尽。有一点需要注意的是`maxmemory`配置的是Redis实际使用的内存量，即`used_memory`，由于有内存碎片的存在，所以实际的内存使用比`used_memory`要大。
Redis可以动态的执行内存的调整：
```
config set maxmemory 6GB
```

**配置内存回收策略**

Redis的内存回收机制主要体现在两个方面上：
> 对过期数据的处理
> 当内存使用情况达到maxmemory时触发内存回收策略


**1. 过期键的删除**
惰性删除：什么时候执行呢？就是在客户端读取带有超时属性的键时，如果已经超过键值设置的过期时间，则删除并返回空。这样做的目的主要是为了节省CPU成本考虑，不需要单独维护TTL链表来处理过期键的删除。但是，如果单独使用这种方式存在一个问题，如果当前的键值永远不再被访问呢？就不删除了吗？那肯定不行，这就会造成内存泄漏的问题。那Redis是怎么解决的呢？Redis提供了一个定时任务的删除机制来做补充。
**2. 定时任务删除**
Redis内部维护了一个定时任务，默认是每秒运行十次。删除的逻辑如下图：
![这里写图片描述](https://img-blog.csdn.net/20180829225531845?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**内存溢出控制策略**
当Redis使用的内存达到上限`maxmemory`后，就会根据`maxmemory-policy`设置的相关策略进行对应的操作，Redis支持一下6中策略：
|策略|说明|
|----|----|
|`noeviction`|默认策略，不会删除任何数据，拒绝所有写入操作并返回客户端错误信息`（error）OOM command not allowed when used memory`|
|`volatile-lru`|只对设置有超时属性的Key根据LRU算法执行删除操作，如果没有可删除的Key，则回退到`noeviction`策略|
|`allkeys-lru`|针对所有的key，根据LRU算法执行删除操作直到回收到足够内存空间|
|`allkeys-random`|随机删除所有的键，直到腾出足够空间|
|`volatile-random`|针对带有过期属性的键，进行删除操作，直到腾出足够空间|
|`volatile-ttl`|根据键值对象的ttl属性，删除最近将要过期数据。如果没有，则回退到noviction策略|

## 内存优化
**Hashtable**

Redis所有的数据存储都是Key-Value的数据类型。整体的结构是Redis自己实现的hashtable，Redis有两个hashtable存在，但是只有其中一个是用来存数据的，另一个hashtable的存在是为了在扩容的时候用的。通过采用渐进式的方式，把旧的hashtable中的数据逐渐的复制到另外一个hashtable中去。为什么采用渐进式呢？因为Redis是单线程的，扩容一直数据的迁移是很耗费时间的，所以迁移的过程是不能对Redis的其他使用造成影响。所以采用渐进式。
![这里写图片描述](https://img-blog.csdn.net/20180902170019292?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
因此，这个hashtable的结构就变得很重要了，hashtable的设计时数组加链表的方式实现，一维是数组结构，二维是一个链表结构，在一维数组中存的是指向链表中第一条数据的指针。
![这里写图片描述](https://img-blog.csdn.net/20180902110711785?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```
//数组结构
struct dictht {
    dictEntry** table; // 二维
    long size; // 第一维数组的长度
    long used; // hash 表中的元素个数
}
//字典实体
struct dictEntry {
    void* key;
    void* val;
    dictEntry* next; // 链接下一个 entry
}
```

**redisObect对象**
Redis中所有的值对象内部定义都是redisObject结构体。结构如下图：
![这里写图片描述](https://img-blog.csdn.net/20180902175713858?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```
struct RedisObject {
    int4 type; // 4bits
    int4 encoding; // 4bits
    int24 lru; // 24bits
    int32 refcount; // 4bytes
    void *ptr; // 8bytes，64-bit system
} robj;...

```
下面详细的说明下每个字段：
|字段名|说明|
|----|----|
|`type`|存储对象的类型，Redis中有5中数据类型：`String`，`List`，`Hash`，`Set`，`Zset`，可以通过 `type {key}`命令查看对象的类型，返回的是值对象类型，所有的key对象都是`String`类型|
|`encoding`|数据存储的Redis中后采用的是那种内部编码格式，这个后边会细讲一下|
|`lru`|记录的是对象被最后一次访问的时间，当配置了`maxmemory`之后，配合LRU算法对相关的key值进行删除，可以通过`object idletime {key}`查看key最近一次被访问的时间。也可以通过`scan + object idletime`命令批量查询那些键长时间没有被使用，从而可以删除长时间没有被使用的键值，减少内存的占用。|
|`refcount`|记录当前对象被引用的次数。根据当前字段来判断该对象时候可回收，当refcount为0时，可安全进行对象的回收，可以使用`object refcount {key}`查看当前对象引用。|
|`*ptr`|与对象的数据内容有关。如果是整数，则直接存储数据（这个地方可以了解下共享对象池，当对象为整数且范围在【0-9999】，会直接存储到共享对象池中），其他类型的数据次字段则代表的是指针。|

**简单动态字符串（simple dynamic string,SDS）**
在Redis中，字符串对象时经常用到的，举个例子：执行一个list的命令，`lpush  queue "redis"  "list" "queue"`，首先会创建`queue`键字符串，然后创建链表对象，链表对象内在包含三个字符串对象。其他的几种结构在存储数据的时候也离不开字符串类型。Redis中字符串的结构也是Redis自己定义的结构，结构图如下：
![这里写图片描述](https://img-blog.csdn.net/20180902190520576?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

```
struct SDS {
    int8 capacity; // 1byte
    int8 len; // 1byte
    int8 flags; // 1byte
    byte[] content; // 内联数组，长度为 capacity
}...
```
SDS有几个特点：

>时间复杂度为O(1)，因为有已知长度，未知长度，字符串长度
>支持安全的二进制数据存储，用于保存字节数组
>内部实现空间预分配机制，降低内存再分配次数
>惰性删除机制，字符串缩减后的空间不释放，作为与分配空间保留

所以字符串在使用的时候，尽量减少追加操作，避免大量的追加操作需要内存重新分配，造成内存碎片率上升。而是尽量使用直接插入。
字符串预分配每次都不是翻倍扩容，空间的预分配规则如下：

- 第一次创建len属性等于数据实际大小，free等于0，不做预分配。
- 修改后，如果已有free空间不够且数据小于1MB，每次与分配一倍容量。
- 修改后如果已有free空间不够且数据大于1MB，每次预分配1MB数据空间。如果数据量太大，也翻一倍的话，很有可能会造成内存不足，所有大于1MB的数据，每次预分配1MB空间，也是基于此原因。 

下面奉上一个简单的例子，比如在Redis中set一个简单点的值，在Redis内存中的结构图如下（参考样例）：
![这里写图片描述](https://img-blog.csdn.net/20180902234609443?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
**编码优化**
Redis有5中不同的存储格式：`String`，`List`，`Hash`，`Set`，`Zset`，但是不同的类型，在存储到Redis中时会有不同的底层数据结构实现。类似集合`ArrayList`的底层实现是数组，而`Linkedlist`的底层实现是链表结构一样，编码不同，则存储是占用的内存以及读写的效率也是不同的。下图是Redis不同结构采用的不同的编码列表：
![这里写图片描述](https://img-blog.csdn.net/20180906211324943?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
**`embstr`与`raw`的区别**
在讲两种编码格式的区别之前，先讲点其他的，现代的计算机的结构上边在CPU和内存之间存在一个缓存结构，用来协调CPU的高效和访存的相对缓慢的矛盾。平时听到的L1 Cache，L2 Cache，L3 Cache就是这个缓存。当CPU要访问内存之前会先在缓存里面找一找看有没有，如果没有，就去内存找，找到之后放到缓存里面。这个缓存的最小单位一般是64字节，一次性缓存连续的64个字节，这个最小的单位称为`缓存行`。
在Redis中，每个`value`对象都有一个`redisObject`对象头，对于Redis的字符串对象，当读取数据时，拿到`*ptr`指针，然后再去找到指向的SDS对象，如果这个对象距离很远，就会影响Redis读取的效率。因此在Redis中设计了一种特殊的编码结构，这种结构就是`embstr`，它把`redisObject`请求头和SDS对象紧紧地挨到一起，然后整体放到一个缓存行中去，这样，在查询的时候就可以直接从缓存中获取到相关的数据，提高了查询的效率。
![这里写图片描述](https://img-blog.csdn.net/2018090821140821?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
在上边也讲了，缓存行一般的长度为64字节，如果想要把对象存到缓存行中，首先整体的长度不得超过64字节，每个请求头`redisObject`占用16字节，而SDS至少有3个自己被占用，同时Redis中会以`\0`来作为结尾的标志，也占用一个字节，因此，留给可输入的数据的长度就成了（64-16-3-1）=44字节，当存储的数据长度超过了44字节，就会变成`raw`的编码形式。
![这里写图片描述](https://img-blog.csdn.net/20180908211542600?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4NTM3MDE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

本篇博客就先讲这么多，目前正处于研究阶段，后续会持续更新。。。

