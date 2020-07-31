

<!-- GFM-TOC -->

* [一、概述](#一概述)
* [二、数据类型](#二数据类型)
    * [STRING](#string)
    * [LIST](#list)
    * [SET](#set)
    * [HASH](#hash)
    * [ZSET](#zset)
* [三、数据结构](#三数据结构)
    * [字典](#字典)
    * [跳跃表](#跳跃表)
* [四、使用场景](#四使用场景)
    * [计数器](#计数器)
    * [缓存](#缓存)
    * [查找表](#查找表)
    * [消息队列](#消息队列)
    * [会话缓存](#会话缓存)
    * [分布式锁实现](#分布式锁实现)
    * [其它](#其它)
* [五、Redis 与 Memcached](#五redis-与-memcached)
    * [数据类型](#数据类型)
    * [数据持久化](#数据持久化)
    * [分布式](#分布式)
    * [内存管理机制](#内存管理机制)
* [六、键的过期时间](#六键的过期时间)
* [七、数据淘汰策略](#七数据淘汰策略)
* [八、持久化](#八持久化)
    * [RDB 持久化](#1. **RDB 快照**(Relational Database)：)
    * [AOF 持久化](#2. **AOF 日志**(append only file))
* [九、事务](#九事务)
* [十、事件](#十事件)
    * [文件事件](#文件事件)
    * [时间事件](#时间事件)
    * [事件的调度与执行](#事件的调度与执行)
* [十一、复制](#十一复制)
    * [连接过程](#连接过程)
    * [主从链](#主从链)
* [十二、Sentinel](#十二sentinel)
* [十三、分片](#十三分片)
* [十四、一个简单的论坛系统分析](#十四一个简单的论坛系统分析)
    * [文章信息](#文章信息)
    * [点赞功能](#点赞功能)
    * [对文章进行排序](#对文章进行排序)
* [参考资料](#参考资料)
<!-- GFM-TOC -->


# 一、概述

Redis（Remote Dictionary Server )，即**远程字典服务**，Redis 是速度非常快的非关系型（NoSQL）内存键值数据库，可以存储**键和五种不同类型的值**之间的映射。

键的类型只能为**string**，值支持五种数据类型：**string、list、set、hash、zset(sorted set：有序集合)**。

Redis 支持很多特性，例如将内存中的数据持久化到硬盘中，使用复制来扩展读性能，使用分片来扩展写性能。

# 二、数据类型

| 数据类型 | 可以存储的值 | 操作 |
| :--: | :--: | :--: |
| STRING(string) | 字符串、整数或者浮点数 | 对整个字符串或者字符串的其中一部分执行操作</br> 对整数和浮点数执行自增或者自减操作 |
| LIST(list) | 列表 | 从两端压入或者弹出元素 </br> 对单个或者多个元素进行修剪，</br> 只保留一个范围内的元素 |
| SET(set) | 无序集合 | 添加、获取、移除单个元素</br> 检查一个元素是否存在于集合中</br> 计算交集、并集、差集</br> 从集合里面随机获取元素 |
| HASH(hash) | 包含键值对的无序散列表 | 添加、获取、移除单个键值对</br> 获取所有键值对</br> 检查某个键是否存在|
| ZSET(zset) | 有序集合（包含键值对） | 添加、获取、删除元素</br> 根据分值范围或者成员来获取元素</br> 计算一个键的排名 |

> [What Redis data structures look like](https://redislabs.com/ebook/part-1-getting-started/chapter-1-getting-to-know-redis/1-2-what-redis-data-structures-look-like/)

## STRING

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/6019b2db-bc3e-4408-b6d8-96025f4481d6.png" width="400"/> </div><br>

```html
> set hello world
OK
> get hello
"world"
> del hello
(integer) 1
> get hello
(nil)
```

## LIST

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/fb327611-7e2b-4f2f-9f5b-38592d408f07.png" width="400"/> </div><br>

```html
> rpush list-key item
(integer) 1
> rpush list-key item2
(integer) 2
> rpush list-key item
(integer) 3

> lrange list-key 0 -1
1) "item"
2) "item2"
3) "item"

> lindex list-key 1
"item2"

> lpop list-key
"item"

> lrange list-key 0 -1
1) "item2"
2) "item"
```

## SET

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/cd5fbcff-3f35-43a6-8ffa-082a93ce0f0e.png" width="400"/> </div><br>

```html
> sadd set-key item
(integer) 1
> sadd set-key item2
(integer) 1
> sadd set-key item3
(integer) 1
> sadd set-key item
(integer) 0

> smembers set-key
1) "item"
2) "item2"
3) "item3"

> sismember set-key item4
(integer) 0
> sismember set-key item
(integer) 1

> srem set-key item2
(integer) 1
> srem set-key item2
(integer) 0

> smembers set-key
1) "item"
2) "item3"
```

## HASH

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/7bd202a7-93d4-4f3a-a878-af68ae25539a.png" width="400"/> </div><br>

```html
> hset hash-key sub-key1 value1
(integer) 1
> hset hash-key sub-key2 value2
(integer) 1
> hset hash-key sub-key1 value1
(integer) 0

> hgetall hash-key
1) "sub-key1"
2) "value1"
3) "sub-key2"
4) "value2"

> hdel hash-key sub-key2
(integer) 1
> hdel hash-key sub-key2
(integer) 0

> hget hash-key sub-key1
"value1"

> hgetall hash-key
1) "sub-key1"
2) "value1"
```

## ZSET

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/1202b2d6-9469-4251-bd47-ca6034fb6116.png" width="400"/> </div><br>

```html
> zadd zset-key 728 member1
(integer) 1
> zadd zset-key 982 member0
(integer) 1
> zadd zset-key 982 member0
(integer) 0

> zrange zset-key 0 -1 withscores
1) "member1"
2) "728"
3) "member0"
4) "982"

> zrangebyscore zset-key 0 800 withscores
1) "member1"
2) "728"

> zrem zset-key member1
(integer) 1
> zrem zset-key member1
(integer) 0

> zrange zset-key 0 -1 withscores
1) "member0"
2) "982"
```

# 三、数据结构

## 字典

dict是Redis服务器中出现最为频繁的复合型数据结构，除hash使用dict之外，整个Redis数据库中所有的key和value也会组成一个全局字典，还有带过期时间的key集合也是一个字典。

zset集合中存储value和score的映射关系也是通过dict结构实现的。

结构：

```c
// 哈希表
typedef struct dictht {
    dictEntry **table;  // 哈希表数组，二维
    long size;          // 哈希表大小
    long used;          // 哈希表已有节点数
} dictht;
​
// 哈希表节点
typedef struct dictEntry {
    void *key;       // 键
    void *val;       // 值
    dictEntry *next; // 指向下一个哈希表节点，形成链表
} dictEntry;
```

dict内部是一个**二维数组**，包含**两个hashtable**。

通常情况下只有一个hashtable是有值的，但是在扩容、缩容时，需要分配新的hashtable，然后进行**渐进式rehash**，此时两个hashtable分别是旧的hashtable和新的hashtable。

在进行rehash时，需要申请新数组，然后迁移所有键值对，在rehash结束后，旧的hashtable被删除，新的hashtable取而代之。这是一个**时间复杂度为O(n)**的操作，所以对于大字典的扩容需要耗费一定时间。为避免阻塞，Redis使用渐进式rehash，多次、渐进地完成迁移，以避免集中式rehash带来的庞大计算量。

![img](https://img2018.cnblogs.com/blog/1779907/201908/1779907-20190831230520901-526181121.png)



从哈希表节点结构dictEntry中可以看到，每一个节点都有一个指向下一个dictEntry的指针，说明Redis中主要通过使用**链表法**解决哈希冲突，即**每一个hashtable中存储的是一个链表**，表中存储指向链表头部元素的指针。

![img](https://img2018.cnblogs.com/blog/1779907/201908/1779907-20190831230546517-369693993.png)

### rehash过程

**1.为字典分配空间**
假设字典中两个hashtable分别为h[0]和h[1]，数据存储在h[0]中。在执行rehash之前，需要**为h[1]分配空间**，这个hashtable的大小取决于需要执行的操作和当前h[0]包含键值对数量即h[0].used：

- 扩展操作：h[1]大小为第一个大于等于h[0].used*2的2的n次方幂

- 收缩操作：h[1]大小为第一个大于等于h[0].used的2的n次方幂


**2.执行rehash**
将保存在**h[0]中的键值对rehash到h[1]上**，rehash指重新计算键的hash值和索引值，然后将键值对放置在h[1]的指定索引位置上

**3.释放空间**
所有键值对迁移完成之后，h[0]变成空表，此时**释放h[0]**，然后将h[1]设置为h[0]，最后在**h[1]位置上新创建一个hashtable**，为下一次rehash做准备。

## 跳跃表

是有序集合的底层实现之一。跳跃表是基于多指针有序链表实现的，可以看成多个有序链表。在查找时，从上层指针开始查找，找到对应的区间之后再到下一层去查找。

插入5的过程：

![img](https://upload-images.jianshu.io/upload_images/6954572-b0b812756bcea352.jpeg)

下图演示了查找 22 的过程。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/0ea37ee2-c224-4c79-b895-e131c6805c40.png" width="600px"/> </div><br>

与红黑树等平衡树相比，跳跃表具有以下优点：

- 插入速度非常快速，因为不需要进行旋转等操作来维护平衡性；
- 更容易实现；
- 支持无锁操作。

# 四、使用场景

## 计数器

可以对 **String 进行自增自减运算**，从而实现计数器功能。

Redis 这种内存型数据库的读写性能非常高，很适合存储频繁读写的计数量。

## 缓存

将**热点数据放到内存**中，设置内存的最大使用量以及淘汰策略来保证缓存的命中率。

## 查找表

例如 DNS 记录就很适合使用 Redis 进行存储。

查找表和缓存类似，也是利用了 **Redis 快速的查找特性**。但是查找表的内容不能失效，而缓存的内容可以失效，因为缓存不作为可靠的数据来源。

## 消息队列

**List 是一个双向链表**，可以通过 **lpush** 和 **rpop** 写入和读取消息

不过最好使用 Kafka、RabbitMQ 等消息中间件。

## 会话缓存

可以使用 Redis 来**统一存储多台应用服务器的会话信息**。

当应用服务器不再存储用户的会话信息，也就不再具有状态，一个用户可以请求任意一个应用服务器，从而更容易实现高可用性以及可伸缩性。

## 分布式锁实现

在分布式场景下，无法使用单机环境下的锁来对多个节点上的进程进行同步。

可以使用 Redis 自带的 **SETNX 命令实现分布式锁**，除此之外，还可以使用官方提供的 RedLock 分布式锁实现。

## 其它

Set 可以实现交集、并集等操作，从而实现共同好友等功能。

ZSet 可以实现有序性操作，从而实现排行榜等功能。

# 五、Redis 与 Memcached

Redis 和 Memcached 都是非关系型**内存key-value数据库**，都是基于内存的数据存储系统。Memcached是高性能分布式内存缓存服务，其本质上就是一个内存key-value数据库。Redis是一个开源的key-value存储系统。与Memcached类似，Redis将大部分数据存储在内存中，支持的数据类型包括：**string、list、set、hash、zset(sorted set：有序集合)**以及基于这些数据类型的相关操作。主要有以下不同：

## 数据类型

Memcached 仅支持字符串类型，而 Redis 支持五种不同的数据类型，可以更灵活地解决问题。

## 数据持久化

Redis 支持两种持久化策略：**RDB 快照**和 **AOF 日志**(append only file)，而 Memcached 不支持持久化。

## 分布式

Memcached 不支持分布式，只能通过在客户端使用一致性哈希来实现分布式存储，这种方式在存储和查询时都需要先在客户端计算一次数据所在的节点。

**Redis Cluster 实现了分布式的支持**。



## 内存管理机制

- 在 Redis 中，并不是所有数据都一直存储在内存中，可以将一些很久没用的 value 交换到磁盘，而 Memcached 的数据则会一直在内存中。

- Memcached 将内存分割成特定长度的块来存储数据，以完全解决内存碎片的问题。但是这种方式会使得内存的利用率不高，例如块的大小为 128 bytes，只存储 100 bytes 的数据，那么剩下的 28 bytes 就浪费掉了。

# 六、键的过期时间

Redis 可以为每个键设置过期时间，当键过期时，会自动删除该键。

对于散列表这种容器，只能为整个键设置过期时间（整个散列表），而不能为键里面的单个元素设置过期时间。

# 七、数据淘汰策略

可以设置内存最大使用量，当内存使用量超出时，会施行数据淘汰策略。

Redis 具体有 6 种淘汰策略：

| 策略 | 描述 |
| :--: | :--: |
| volatile-lru | 从**已设置过期时间的数据集**中挑选**最近最少使用**的数据淘汰 |
| volatile-ttl | 从**已设置过期时间的数据集**中挑选**将要过期**的数据淘汰 |
|volatile-random | 从**已设置过期时间的数据集**中挑选**任意选择**数据淘汰 |
| allkeys-lru | 从**所有数据集**中挑选**最近最少使用**的数据淘汰 |
| allkeys-random | 从**所有数据集**中**任意选择**数据进行淘汰 |
| noeviction | 禁止驱逐数据 |

作为内存数据库，出于对性能和内存消耗的考虑，Redis 的淘汰算法实际实现上并非针对所有 key，而是抽样一小部分并且从中选出被淘汰的 key。

使用 Redis 缓存数据时，为了提高缓存命中率，需要保证缓存数据都是热点数据。可以将内存最大使用量设置为热点数据占用的内存量，然后启用 allkeys-lru 淘汰策略，将最近最少使用的数据淘汰。

Redis 4.0 引入了 volatile-lfu 和 allkeys-lfu 淘汰策略，LFU 策略通过统计访问频率，将访问频率最少的键值对淘汰。

# 八、持久化

Redis 是内存型数据库，为了保证数据在断电后不会丢失，需要将内存中的数据持久化到硬盘上。

### 1. **RDB 快照**(Relational Database)：

通过拍摄快照方式进行数据的持久化，将某个时间节点的数据存储到一个**rdb文件**中，当再次启动redis服务时，会从这个rdb文件加载数据。

**（1）触发方式：**

- **save**：在redis运行中，我们可以显示的发送一条save命令来拍摄快照。save命令是**阻塞命令**，也就是当服务器接收了一条save命令之后就会开始拍摄快照，在此期间不会再去处理其他的请求，其他请求会被挂起直到备份结束。
- **bgsave**：使用bgsave命令也是立即拍摄快照，有别于save命令，bgsave并不是一条阻塞命令，而是**fork一个子线程**，然后这个子线程负责备份操作。而父进程继续处理客户端的请求，这样就不会造成阻塞了。当数据集比较大的时候，fork的过程是非常耗时的，频繁执行影响性能。
- flushall：清空命令也会触发持久化操作，但dump.rdb文件中是空的，无意义
- **shutdown**：当我们只想shutdown命令的时候。服务器会自动发送一条save命令来完成快照操作。并在完成备份操作后关闭服务器
- 配置 redis.conf 文件自动触发

**（2）恢复快照中的数据**

　　将**备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务**即可，redis就会自动加载文件数据至内存了。Redis 服务器在载入 RDB 文件期间，会一直处于阻塞状态，直到载入工作完成为止。

　　获取 redis 的安装目录可以使用 config get dir 命令。

### 2. **AOF 日志**(append only file)

以独立日志的方式将**写命令**添加到 AOF 文件的末尾。写入的内容直接是文本协议格式，重启时再重新执行AOF文件中的命令达到恢复数据的目的，解决了数据持久化实时性问题。

**（1）设置同步选项**

使用 AOF 持久化需要设置同步选项，从而确保写命令同步到磁盘文件上的**时机**。这是因为对文件进行写入并不会马上将内容同步到磁盘上，而是先存储到缓冲区，然后由操作系统决定什么时候同步到磁盘。有以下同步选项：

| 选项 | 同步频率 |
| :--: | :--: |
| always | 每个写命令都同步 |
| everysec | 每秒同步一次 |
| no | 让操作系统来决定何时同步 |

- always 选项会严重减低服务器的性能；
- everysec 选项比较合适，可以保证系统崩溃时只会丢失一秒左右的数据，并且 Redis 每秒执行一次同步对服务器性能几乎没有任何影响；
- no 选项并不能给服务器性能带来多大的提升，而且也会增加系统崩溃时数据丢失的数量。

**（2）ReWrite重写机制**：

AOF采用文件追加的方式，将导致文件越来越大，故新增重写机制。当AOF文件大小超过所设置的阈值时，Redis将启动AOF内容压缩，只保留可以恢复数据的最小指令集。

Redis 提供了 bgrewriteaof 指令用于对 AOF 日志进行瘦身（重写）。其原理就是开辟一个子进程对内存进行遍历转换成一系列 Redis 的操作指令，序列化到一个新的 AOF 日志文件中。序列化完毕后再将操作期间发生的增量AOF 日志追加到这个新的 AOF 日志文件中，追加完毕后就立即替代旧的 AOF 日志文件了，瘦身工作就完成了。

# 九、事务

一个事务包含了**多个命令**，服务器在执行事务期间，不会改去执行其它客户端的命令请求。

事务中的多个命令被**一次性发送给服务器**，而不是一条一条发送，这种方式被称为**流水线**，它可以减少客户端与服务器之间的网络通信次数从而提升性能。

Redis 最简单的事务实现方式是使用 MULTI(multi) 和 EXEC (exec)命令将事务操作包围起来。MULTI 标记一个事务块的开始；EXEC 执行所有事务块内的命令。

# 十、事件

Redis 服务器是一个事件驱动程序。

## 文件事件

服务器通过**套接字(Socket)**与客户端或者其它服务器进行通信，**文件事件就是对套接字操作的抽象**，当一个socket准备好执行accept， read，write等操作时，就会产生一个文件事件。

Redis 基于 **Reactor** 模式开发了自己的网络事件处理器，使用 **I/O 多路复用程序**来同时监听多个套接字，并将到达的事件传送给**文件事件分派器**，分派器会根据套接字产生的事件类型调用相应的**事件处理器**。

![image-20200731202109532](C:\Users\zblz2\AppData\Roaming\Typora\typora-user-images\image-20200731202109532.png)

尽管多个文件事件可能会并发的出现，但I/O多路复用程序总是会将所有产生的事件socket都放到一个**队列**中，然后通过这个队列，以**有序**，**同步**，每次**一个**socket的方式向**文件事件分派器**传送。当上一个socket处理完毕后（该socket为事件所关联的**事件处理器**执行完毕），I/O多路复用程序才会继续向文件事件分派器传送下一个socket。

## 时间事件

服务器有一些操作需要**在给定的时间点执行**，**时间事件是对这类定时操作的抽象**。

时间事件又分为：

- **定时**事件：是让一段程序在指定的时间之内执行一次；
- **周期性**事件：是让一段程序每隔指定时间就执行一次。

一个时间事件主要由以下三个属性组成：

- **id**：服务器为时间事件创建的全局唯一ID。ID号按从小到大的顺序递增，新事件的ID号比旧事件的ID号要大。
- **when**：毫秒精度的**UNIX时间戳**，记录了时间事件的到达时间。
- **timeProc**：**时间事件处理器**，一个函数。当时间事件到达时，服务器就会调用相应的处理器来处理事件。

Redis 将所有时间事件都放在一个**无序链表**中，通过遍历整个链表查找出已到达的时间事件，并调用相应的事件处理器。

## 事件的调度与执行

服务器需要不断监听文件事件的套接字才能得到待处理的文件事件，但是不能一直监听，否则时间事件无法在规定的时间内执行，因此监听时间应该根据距离现在最近的时间事件来决定。

事件调度与执行由 **aeProcessEvents** 函数负责，伪代码如下：

```python
def aeProcessEvents():
    # 获取到达时间离当前时间最接近的时间事件
    time_event = aeSearchNearestTimer()
    # 计算最接近的时间事件距离到达还有多少毫秒
    remaind_ms = time_event.when - unix_ts_now()
    # 如果事件已到达，那么 remaind_ms 的值可能为负数，将它设为 0
    if remaind_ms < 0:
        remaind_ms = 0
    # 根据 remaind_ms 的值，创建 timeval
    timeval = create_timeval_with_ms(remaind_ms)
    # 阻塞并等待文件事件产生，最大阻塞时间由传入的 timeval 决定
    aeApiPoll(timeval)
    # 处理所有已产生的文件事件
    procesFileEvents()
    # 处理所有已到达的时间事件
    processTimeEvents()
```

将 aeProcessEvents 函数置于一个循环里面，加上初始化和清理函数，就构成了 Redis 服务器的主函数，伪代码如下：

```python
def main():
    # 初始化服务器
    init_server()
    # 一直处理事件，直到服务器关闭为止
    while server_is_not_shutdown():
        aeProcessEvents()
    # 服务器关闭，执行清理操作
    clean_server()
```

从事件处理的角度来看，服务器运行流程如下：

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/c0a9fa91-da2e-4892-8c9f-80206a6f7047.png" width="350"/> </div><br>

# 十一、复制

通过使用 **slaveof host port** 命令来让一个服务器成为另一个服务器(master server)的从服务器(slave server)。

一个从服务器只能有一个主服务器，并且不支持主主复制。

## 连接过程

1. 主服务器创建**快照文件**，发送给从服务器，并在发送期间使用**缓冲区**记录执行的写命令。快照文件发送完毕之后，开始向从服务器发送存储在缓冲区中的写命令；

2. 从服务器**丢弃所有旧数据**，载入主服务器发来的快照文件，之后从服务器开始接受主服务器发来的写命令；

3. 主服务器每执行一次**写命令**，就向从服务器发送相同的写命令。

## 主从链

随着负载不断上升，主服务器可能无法很快地更新所有从服务器，或者重新连接和重新同步从服务器将导致系统超载。为了解决这个问题，可以创建一个中间层来分担主服务器的复制工作。**中间层的服务器**是最上层服务器的从服务器，又是最下层服务器的主服务器。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/395a9e83-b1a1-4a1d-b170-d081e7bb5bab.png" width="600"/> </div><br>

# 十二、Sentinel

**Sentinel**（哨兵）可以监听集群中的服务器，并在主服务器进入下线状态时，自动**从从服务器中选举出新的主服务器**。

# 十三、分片

分片是将**数据划分为多个部分**的方法，可以将数据**存储到多台机器**里面，这种方法在解决某些问题时可以获得线性级别的性能提升。

假设有 4 个 Redis 实例 R0，R1，R2，R3，还有很多表示用户的键 user:1，user:2，... ，有不同的方式来选择一个指定的键存储在哪个实例中。

- 最简单的方式是范围分片，例如用户 id 从 0\~1000 的存储到实例 R0 中，用户 id 从 1001\~2000 的存储到实例 R1 中，等等。但是这样需要维护一张映射范围表，维护操作代价很高。
- 还有一种方式是哈希分片，使用 CRC32 哈希函数将键转换为一个数字，再对实例数量求模就能知道应该存储的实例。

根据执行分片的位置，可以分为三种分片方式：

- **客户端**分片：客户端使用一致性哈希等算法决定键应当分布到哪个节点。
- **代理**分片：将客户端请求发送到代理上，由代理转发请求到正确的节点上。
- **服务器**分片：Redis Cluster。

# 十四、一个简单的论坛系统分析

该论坛系统功能如下：

- 可以发布文章；
- 可以对文章进行点赞；
- 在首页可以按文章的发布时间或者文章的点赞数进行排序显示。

## 文章信息

文章包括标题、作者、赞数等信息，在关系型数据库中很容易构建一张表来存储这些信息，在 Redis 中可以使用 HASH 来存储每种信息以及其对应的值的映射。

Redis 没有关系型数据库中的表这一概念来将同种类型的数据存放在一起，而是使用命名空间的方式来实现这一功能。键名的前面部分存储命名空间，后面部分的内容存储 ID，通常使用 : 来进行分隔。例如下面的 HASH 的键名为 article:92617，其中 article 为命名空间，ID 为 92617。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/7c54de21-e2ff-402e-bc42-4037de1c1592.png" width="400"/> </div><br>

## 点赞功能

当有用户为一篇文章点赞时，除了要对该文章的 votes 字段进行加 1 操作，还必须记录该用户已经对该文章进行了点赞，防止用户点赞次数超过 1。可以建立文章的已投票用户集合来进行记录。

为了节约内存，规定一篇文章发布满一周之后，就不能再对它进行投票，而文章的已投票集合也会被删除，可以为文章的已投票集合设置一个一周的过期时间就能实现这个规定。

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/485fdf34-ccf8-4185-97c6-17374ee719a0.png" width="400"/> </div><br>

## 对文章进行排序

为了按发布时间和点赞数进行排序，可以建立一个文章发布时间的有序集合和一个文章点赞数的有序集合。（下图中的 score 就是这里所说的点赞数；下面所示的有序集合分值并不直接是时间和点赞数，而是根据时间和点赞数间接计算出来的）

<div align="center"> <img src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/f7d170a3-e446-4a64-ac2d-cb95028f81a8.png" width="800"/> </div><br>

# 参考资料

- Carlson J L. Redis in Action[J]. Media.johnwiley.com.au, 2013.
- [黄健宏. Redis 设计与实现 [M]. 机械工业出版社, 2014.](http://redisbook.com/index.html)
- [REDIS IN ACTION](https://redislabs.com/ebook/foreword/)
- [Skip Lists: Done Right](http://ticki.github.io/blog/skip-lists-done-right/)
- [论述 Redis 和 Memcached 的差异](http://www.cnblogs.com/loveincode/p/7411911.html)
- [Redis 3.0 中文版- 分片](http://wiki.jikexueyuan.com/project/redis-guide)
- [Redis 应用场景](http://www.scienjus.com/redis-use-case/)
- [Using Redis as an LRU cache](https://redis.io/topics/lru-cache)






<div align="center"><img width="320px" src="https://cs-notes-1256109796.cos.ap-guangzhou.myqcloud.com/githubio/公众号二维码-2.png"></img></div>
