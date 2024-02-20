# Redis是什么？

[Redis](https://redis.io/)是一个基于 C 语言开发的开源数据库（遵守BSD协议），数据存储在**内存**中（内存数据库），读写速度非常快，被广泛应用于缓存方向。并且，Redis 存储的是 **KV 键值对数据**。

为了满足不同的业务场景，Redis 内置了多种数据类型实现（比如 **String、List、Hash、Set、Sorted Set、Bitmap、HyperLogLog、GEO**）。

它内置了**复制（Replication）**、**LUA 脚本（Lua scripting）**、**LRU 缓存淘汰（LRU eviction）**、**事务（Transactions）**、**发布/订阅**、**流技术**和不同级别的**磁盘持久化（persistence）**功能。

并提供了**主从模式**、 **Redis Sentinel（哨兵）**和 **Redis Cluster（集群）**保证缓存的高可用性（High availability）。

[英文官网 https://redis.io/](https://redis.io/)

[中文网站 https://redis.com.cn/documentation.html](https://redis.com.cn/documentation.html)

[官网下载地址 https://redis.io/download/](https://redis.io/download/)

[源码地址 https://github.com/redis/redis](https://github.com/redis/redis)

[在线测试 https://try.redis.io/](https://try.redis.io/)

[命令参考 http://doc.redisfans.com/](http://doc.redisfans.com/)

# Redis能干嘛？

## Redis用途

1. 分布式缓存，挡在mysql之前的带刀护卫。
2. 内存存储和持久化（RDB+AOF），redis支持异步将内存中的数据写到硬盘上，同时不影响继续服务
3. 高可用架构（单机、主从、哨兵、集群）
4. 缓存穿透、击穿、雪崩
5. 分布式锁
6. 队列
7. 排行榜+点赞
8. 。。。。。。

## Redis优势

1. 性能极高，redis读取速度110000次/秒，写的速度81000次/秒
2. redis数据类型丰富，不仅仅支持简单的key-value类型的数据，同时还提供list、set、zset、hash等数据结构
3. redis支持数据的持久化，可以将内存的数据持久化到磁盘中，重启的时候再次加载使用
4. redis支持数据的备份，即master-slave模式的数据备份

# Redis迭代演化

![image-20231219220527144](pictures/image-20231219220527144.png)

[历史版本源码 https://download.redis.io/releases/](https://download.redis.io/releases/) 版本号如果第二位是奇数，则为非稳定版本如2.7、2.9

[历史版本新特性 https://github.com/redis/redis/releases](https://github.com/redis/redis/releases)

## Redis7.0新特性todo



# Redis安装与卸载

**Redis安装**

```
0、gcc编译环境确认
gcc是linux下的一个编译程序，是C程序的编译工具。
查看gcc版本 gcc -v
安装gcc yum -y install gcc-c++


1、下载redis到/opt目录下
cd /opt
wget https://download.redis.io/releases/redis-7.0.0.tar.gz

2、解压
tar -zxvf redis-7.0.0.tar.gz

3、进入目录
cd redis-7.0.0

4、执行make命令
make && make install

5、查看默认安装目录 /usr/local/bin  /usr/local类似于windows系统中的C:\Program Files
redis-benchmark：性能测试工具
redis-check-aof：修复有问题的AOF文件
redis-check-dump：修复有问题的dump.rdb文件
redis-cli：客户端
redis-sentinel：redis集群使用
redis-server：redis服务启动命令

6、修改redis.conf配置文件
vim /opt/redis-7.0.0/redis.conf
默认daemonize no      改为  daemonize yes  通过后台启动
默认protected-mode  yes      改为  protected-mode no  安全模式，开启外部无法连接
默认bind 127.0.0.1      改为  直接注释掉或改成本机IP地址，否则影响远程IP连接
添加redis密码      改为 requirepass 你自己设置的密码

7、启动服务
cd /usr/local/bin
redis-server /opt/redis-7.0.0/redis.conf

8、连接服务
redis-cli -a password

9、校验是否启动成功
127.0.0.1:6379> ping
PONG

10、关闭服务
redis-cli -a password shutdown
```

**Redis卸载**

```
1、关闭服务
redis-cli -a password shutdown

2、删除/usr/local/lib下关于redis的文件
ls -l /usr/local/bin/redis-*
rm -rf /usr/local/bin/redis-*
```

# Redis 为什么这么快？

1. Redis 基于内存，内存的访问速度是磁盘的上千倍。
2. Redis 基于 Reactor 模式设计开发了一套高效的事件处理模型，主要是**单线程事件循环**(单线程的话就能避免多线程的频繁上下文切换问题)和 **非阻塞的IO多路复用**。
3. Redis 内置了多种优化过后的数据结构实现，性能非常高。
4. Redis 是用C语言实现的

![why-redis-so-fast](pictures/why-redis-so-fast-d3507ae8.png)

# 为什么要用 Redis/为什么要用缓存？

**1、高性能**
假如用户第一次访问数据库中的某些数据的话，这个过程是比较慢，毕竟是从硬盘中读取的。但是，如果说，用户访问的数据属于高频数据并且不会经常改变的话，那么我们就可以很放心地将该用户访问的数据存在缓存中。
这样有什么好处呢？ 那就是保证用户下一次再访问这些数据的时候就可以直接从缓存中获取了。操作缓存就是直接操作内存，所以速度相当快。

**2、高并发**

一般像 MySQL 这类的数据库的 QPS 大概都在 1w 左右（4 核 8g） ，但是使用 Redis 缓存之后很容易达到 10w+，甚至最高能达到 30w+（就单机 Redis 的情况，Redis 集群的话会更高）。

> QPS（Query Per Second）：服务器每秒可以执行的查询次数；

由此可见，直接操作缓存能够承受的数据库请求数量是远远大于直接访问数据库的，所以我们可以考虑把数据库中的部分数据转移到缓存中去，这样用户的一部分请求会直接到缓存这里而不用经过数据库。进而，我们也就提高了系统整体的并发。

# 基本数据结构

[Redis 数据类型](https://redis.io/docs/data-types/)

`help @数据类型` 查看帮助

Redis 共有 5 种基本数据结构：String（字符串）、List（列表）、Set（集合）、Hash（散列）、Zset（有序集合）。

这 5 种数据结构是直接提供给用户使用的，是数据的保存形式，其底层实现主要依赖这 8 种数据结构：简单动态字符串（SDS）、LinkedList（双向链表）、Dict（哈希表/字典）、SkipList（跳跃表）、Intset（整数集合）、ZipList（压缩列表）、QuickList（快速列表）。

| String | List                         | Hash          | Set          | Zset              |
| :----- | :--------------------------- | :------------ | :----------- | :---------------- |
| SDS    | LinkedList/ZipList/QuickList | Dict、ZipList | Dict、Intset | ZipList、SkipList |

Redis 3.2 之前，List 底层实现是 LinkedList 或者 ZipList。 Redis 3.2 之后，引入了 LinkedList 和 ZipList 的结合 QuickList，List 的底层实现变为 QuickList。从 Redis 7.0 开始， ZipList 被 ListPack 取代。

可以在 Redis 官网上找到 Redis 数据结构非常详细的介绍：

- [Redis Data Structures](https://redis.com/redis-enterprise/data-structures/)
- [Redis Data types tutorial](https://redis.io/docs/manual/data-types/data-types-tutorial/)

## String（字符串）

String 是 Redis 中最简单同时也是最常用的一个数据结构。

String 是一种二进制安全的数据结构，可以用来存储任何类型的数据比如字符串、整数、浮点数、图片（图片的 base64 编码或者解码或者图片的路径）、序列化后的对象。Redis中字符串value最多可以是**512M**。

虽然 Redis 是用 C 语言写的，但是 Redis 并没有使用 C 的字符串表示，而是自己构建了一种 **简单动态字符串**（Simple Dynamic String，**SDS**）。相比于 C 的原生字符串，Redis 的 SDS 不光可以保存文本数据还可以保存二进制数据，并且获取字符串长度复杂度为 O(1)（C 字符串为 O(N)）,除此之外，Redis 的 SDS API 是安全的，不会造成缓冲区溢出。

### **常用命令**

https://redis.io/commands/?group=string

| 命令                             | 介绍                                              |
| :------------------------------- | :------------------------------------------------ |
| SET key value                    | 设置指定 key 的值                                 |
| SETNX key value                  | 只有在 key 不存在时设置 key 的值 set if not exist |
| SETEX key seconds value          | 设置指定 key 的值并设置过期时间 set with expire   |
| GET key                          | 获取指定 key 的值                                 |
| MSET key1 value1 key2 value2 …   | 设置一个或多个指定 key 的值                       |
| MSETNX key1 value1 key2 value2 … | 只有在都不存在时，设置一个或多个指定 key 的值     |
| MGET key1 key2 ...               | 获取一个或多个指定 key 的值                       |
| STRLEN key                       | 返回 key 所储存的字符串值的长度                   |
| APPEND key str                   | 在指定key后追加str                                |
| INCR key                         | 将 key 中储存的数字值增一                         |
| INCRBY key increment             | 将 key 所储存的值加上给定的增量值（increment）    |
| DECR key                         | 将 key 中储存的数字值减一                         |
| DECRBY key decrement             | key 所储存的值减去给定的减量值（decrement）       |
| GETSET key value                 | 设置指定 key 的值，并返回之前的值                 |
| EXISTS key（通用）               | 判断指定 key 是否存在                             |
| DEL key（通用）                  | 删除指定的 key                                    |
| EXPIRE key seconds（通用）       | 给指定 key 设置过期时间                           |

### 命令详解

**SET**

SET key value [NX | XX] [GET] [EX seconds | PX milliseconds | EXAT unix-time-seconds | PXAT unix-time-milliseconds | KEEPTTL]

- `EX` *seconds*：设置指定过期时间，单位为秒
- `PX` *milliseconds*：设置指定过期时间，单位为毫秒
- `EXAT` *timestamp-seconds*：设置指定unix时间戳作为过期时间，单位为秒 `System.currentTimeMillis()/1000`
- `PXAT` timestamp-milliseconds：设置指定unix时间戳作为过期时间，单位为毫秒
- `NX`：仅在key不存在的情况下设置value
- `XX`：仅在key存在的情况下设置value
- `KEEPTTL`：保留设置前指定的生存时间
- `GET`：返回以 key 存储的旧字符串，如果 key 不存在，返回 nil。如果键上存储的值不是字符串，则返回错误并中止 SET。

### **应用场景**

**需要存储常规数据的场景**

- 举例：缓存 session、token、图片地址、序列化后的对象(相比较于 Hash 存储更节省内存)。
- 相关命令：`SET`、`GET`。

**需要计数的场景**

- 举例：用户单位时间的请求数（简单限流可以用到）、页面单位时间的访问数。
- 相关命令：`SET`、`GET`、 `INCR`、`DECR` 。

**分布式锁**

利用 `SETNX key value` 命令可以实现一个最简易的分布式锁（存在一些缺陷，通常不建议这样实现分布式锁）。

## List（列表）

Redis 的 List 的实现为一个 **双向链表**，容量是**2的32次方-1**，即可以支持反向查找和遍历，更方便操作，不过带来了部分额外的内存开销。

### **常用命令**

https://redis.io/commands/?group=list

| **命令**                                   | **介绍**                                        |
| :----------------------------------------- | ----------------------------------------------- |
| RPUSH key value1 value2 ...                | 在指定列表的尾部（右边）添加一个或多个元素      |
| LPUSH key value1 value2 ...                | 在指定列表的头部（左边）添加一个或多个元素      |
| LSET key index value                       | 将指定列表索引 index 位置的值设置为 value       |
| LPOP key                                   | 移除并获取指定列表的第一个元素(最左边)          |
| RPOP key                                   | 移除并获取指定列表的最后一个元素(最右边)        |
| LINDEX key index                           | 按照索引下标获取元素                            |
| LLEN key                                   | 获取列表元素数量                                |
| LRANGE key start end                       | 获取列表 start 和 end 之间 的元素 0到-1代表所有 |
| LREM key num value                         | 删除指定key中num个value值                       |
| LTRIM key sindex eindex                    | 截取指定范围key并赋值给该key                    |
| RPOPLPUSH key1 key2                        | 从key1右端弹出插入到key2左端                    |
| LINSERT key before/after 已有值 插入的新值 | 在指定key的已有值的前/后插入新值                |

![img](pictures/redis-list.png)

### 应用场景

**信息流展示**

- 举例：最新文章、最新动态。
- 相关命令：`LPUSH`、`LRANGE`。

将文章/动态id存入redis，可以利用`LRANGE`进行分页查询

**消息队列**

Redis List 数据结构可以用来做消息队列，只是功能过于简单且存在很多缺陷，不建议这样做。

相对来说，Redis 5.0 新增加的一个数据结构 `Stream` 更适合做消息队列一些，只是功能依然非常简陋。和专业的消息队列相比，还是有很多欠缺的地方比如消息丢失和堆积问题不好解决。

**流量削峰**

## Hash（哈希）

Redis 中的 Hash 是一个 String 类型的 field-value（键值对） 的映射表，特别适合用于存储对象，后续操作的时候，你可以直接修改这个对象中的某些字段的值。

Hash 类似于 JDK1.8 前的 `HashMap`，内部实现也差不多(数组 + 链表)。不过，Redis 的 Hash 做了更多优化。

### 常用命令

https://redis.io/commands/?group=hash

| 命令                                      | 介绍                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| HSET key field value                      | 设置指定哈希表中指定字段的值                                 |
| HGET key field                            | 获取指定哈希表中指定字段的值                                 |
| HSETNX key field value                    | 只有指定字段不存在时设置指定字段的值                         |
| HMSET key field1 value1 field2 value2 ... | 同时将一个或多个 field-value (域-值)对设置到指定哈希表中     |
| HMGET key field1 field2 ...               | 获取指定哈希表中一个或者多个指定字段的值                     |
| HGETALL key                               | 获取指定哈希表中所有的键值对                                 |
| HDEL key field1 field2 ...                | 删除一个或多个哈希表字段                                     |
| HEXISTS key field                         | 查看指定哈希表中指定的字段是否存在                           |
| HKEYS key                                 | 获取指定哈希表所有field                                      |
| HVALS key                                 | 获取指定哈希表所有value                                      |
| HLEN key                                  | 获取指定哈希表中字段的数量                                   |
| HINCRBY key field increment               | 对指定哈希中的指定字段做运算操作（正数为加，负数为减）       |
| HINCRBYFLOAT key field increment          | 对指定哈希中的指定字段做浮点数运算操作（正数为加，负数为减） |

**HINCRBY**

```
> HSET key n1 100
(integer) 1
> HINCRBY key n1 200
(integer) 300
> HGET key n1
"300"
```

### 应用场景

**对象数据存储场景**

- 举例：用户信息、商品信息、文章信息、购物车信息。
- 相关命令：`HSET` （设置单个字段的值）、`HMSET`（设置多个字段的值）、`HGET`（获取单个字段的值）、`HMGET`（获取多个字段的值）。

## Set（集合）

Redis 中的 Set 类型是一种**无序集合**，集合中的元素没有先后顺序但都**唯一**，有点类似于 Java 中的 `HashSet` 。当你需要存储一个列表数据，又不希望出现重复数据时，Set 是一个很好的选择，并且 Set 提供了**判断某个元素是否在**一个 Set 集合内的重要接口，这个也是 List 所不能提供的。Set集合是通过哈希表实现的，所以添加、删除、查找的复杂度都是**O(1)**。

可以基于 Set 轻易实现**交集、并集、差集**的操作，比如你可以将一个用户所有的关注人存在一个集合中，将其所有粉丝存在一个集合。这样的话，Set 可以非常方便的实现如共同关注、共同粉丝、共同喜好等功能。这个过程也就是求交集的过程。

### 常用命令

https://redis.io/commands/?group=set

| 命令                                  | 介绍                                         |
| ------------------------------------- | -------------------------------------------- |
| SADD key member1 member2 ...          | 向指定集合添加一个或多个元素                 |
| SMEMBERS key                          | 获取指定集合中的所有元素                     |
| SISMEMBER key member                  | 判断指定元素是否在指定集合中                 |
| SCARD key                             | 获取指定集合的元素数量                       |
| SREM key member1 member2 ...          | 删除指定集合中的元素                         |
| SPOP key [count]                      | 随机移除并获取指定集合中一个或多个元素，删除 |
| SRANDMEMBER key [count]               | 随机获取指定集合中指定数量的元素，不删除     |
| SMOVE key1 key2 k1value               | 将key1中已存在的某个值迁移到key2中           |
|                                       |                                              |
| SINTER key1 key2 ...                  | 获取给定所有集合的交集(我有你也有)           |
| SINTERSTORE destination key1 key2 ... | 将给定所有集合的交集存储在 destination 中    |
| SUNION key1 key2 ...                  | 获取给定所有集合的并集(我有加你有)           |
| SUNIONSTORE destination key1 key2 ... | 将给定所有集合的并集存储在 destination 中    |
| SDIFF key1 key2 ...                   | 获取给定所有集合的差集(我有你没有)           |
| SDIFFSTORE destination key1 key2 ...  | 将给定所有集合的差集存储在 destination 中    |

### 应用场景

**需要存放的数据不能重复的场景**

- 举例：网站 UV 统计（数据量巨大的场景还是 `HyperLogLog`更适合一些）、文章点赞、动态点赞等场景。
- 相关命令：`SCARD`（获取集合数量）。

**需要获取多个数据源交集、并集和差集的场景**

举例：共同好友(交集)、共同粉丝(交集)、共同关注(交集)、好友推荐（差集）、音乐推荐（差集）、订阅号推荐（差集+交集） 等场景。

相关命令：`SINTER`（交集）、`SINTERSTORE` （交集）、`SUNION` （并集）、`SUNIONSTORE`（并集）、`SDIFF`（差集）、`SDIFFSTORE` （差集）。

**需要随机获取数据源中的元素的场景**

- 举例：抽奖系统、随机点名等场景。
- 相关命令：`SPOP`（随机获取集合中的元素并移除，适合不允许重复中奖的场景）、`SRANDMEMBER`（随机获取集合中的元素，适合允许重复中奖的场景）。

## Zset（有序集合）

Sorted Set 类似于 Set，但和 Set 相比，Sorted Set 增加了一个权重参数 `score`，使得集合中的元素能够按 `score` 进行有序排列，还可以通过 `score` 的范围来获取元素的列表。有点像是 Java 中 `HashMap` 和 `TreeSet` 的结合体。Zset集合是通过哈希表实现的，所以添加、删除、查找的复杂度都是**O(1)**。

### **常用命令**

https://redis.io/commands/?group=sorted-set

| 命令                                                        | 介绍                                                         |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| ZADD key score1 member1 score2 member2 ...                  | 向指定有序集合添加一个或多个元素                             |
| ZRANGE key start end [WITHSCORES]                           | 获取指定有序集合 start 和 end 之间的元素（score 从低到高）   |
| ZREVRANGE key start end [WITHSCORES]                        | 获取指定有序集合 start 和 end 之间的元素（score 从高到底）   |
| ZRANGEBYSCORE key min max [WITHSCORES] [limit offset count] | 获取指定分数范围内的元素，(代表不包含                        |
| ZCARD KEY                                                   | 获取指定有序集合的元素数量                                   |
| ZSCORE key member                                           | 获取指定有序集合中指定元素的 score 值                        |
| ZREM key member1 member2 ...                                | 删除有序集合中的元素                                         |
| ZINCRBY key increment member                                | 指定key中的member增加分数                                    |
| ZCOUNT key min max                                          | 获取指定分数范围内的元素个数                                 |
| ZRANK key member                                            | 获取指定有序集合中指定元素的排名(score 从小到大排序)         |
| ZREVRANK key member                                         | 获取指定有序集合中指定元素的排名(score 从大到小排序)         |
|                                                             |                                                              |
| ZINTERSTORE destination numkeys key1 key2 ...               | 将给定所有有序集合的交集存储在 destination 中，对相同元素对应的 score 值进行 SUM 聚合操作，numkeys 为集合数量 |
| ZUNIONSTORE destination numkeys key1 key2 ...               | 求并集，其它和 ZINTERSTORE 类似                              |
| ZDIFFSTORE destination numkeys key1 key2 ...                | 求差集，其它和 ZINTERSTORE 类似                              |

**ZRANGEBYSCORE**

```
> zadd z 1 a 2 b 3 c
(integer) 3
> zrangebyscore z 1 3
a b c
> zrangebyscore z 1 (3
a b
> zrangebyscore z (1 3
b c
> zrangebyscore z 1 3 limit 1 1
b
```

### 应用场景

**需要随机获取数据源中的元素根据某个权重进行排序的场景**

举例：各种排行榜比如直播间送礼物的排行榜、朋友圈的微信步数排行榜、王者荣耀中的段位排行榜、话题热度排行榜等等。

相关命令：`ZRANGE` (从小到大排序)、 `ZREVRANGE` （从大到小排序）、`ZREVRANK` (指定元素排名)。

**需要存储的数据有优先级或者重要程度的场景** 比如优先级任务队列。

- 举例：优先级任务队列。
- 相关命令：`ZRANGE` (从小到大排序)、 `ZREVRANGE` （从大到小排序）、`ZREVRANK` (指定元素排名)。

# 特殊数据结构

除了 5 种基本的数据结构之外，Redis 还支持 3 种特殊的数据结构：Bitmap、HyperLogLog、GEO。

## Bitmap（位图）

Bitmap 存储的是连续的二进制数字（0 和 1），通过 Bitmap, 只需要一个 bit 位来表示某个元素对应的值或者状态，key 就是对应元素本身 。我们知道 8 个 bit 可以组成一个 byte，所以 Bitmap 本身会极大的节省储存空间。**用String类型作为底层数据结构实现的一种统计二值状态的数据结构，位图的本质是数组**，最大支持位数是**2的32次方**位，使用512M内存就可以存储多大42.9亿的字节信息

可以将 Bitmap 看作是一个存储二进制数字（0 和 1）的数组，数组中每个元素的下标叫做 offset（偏移量）。

![img](pictures/image-20220720194154133.png)

### **常用命令**

| 命令                                  | 介绍                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| SETBIT key offset value               | 设置指定 offset 位置的值                                     |
| GETBIT key offset                     | 获取指定 offset 位置的值                                     |
| STRLEN key                            | 获取指定 key 占用的字节数                                    |
| BITCOUNT key start end                | 获取 start 和 end 之前值为 1 的元素个数                      |
| BITOP operation destkey key1 key2 ... | 对一个或多个 Bitmap 进行运算，可用运算符operation有 AND, OR, XOR 以及 NOT，destkey为输出的key |

**基础操作**

```
> SETBIT mykey 7 1
(integer) 0
> SETBIT mykey 7 0
(integer) 1
> GETBIT mykey 7
(integer) 0
> SETBIT mykey 6 1
(integer) 0
> SETBIT mykey 8 1
(integer) 0
> BITCOUNT mykey
(integer) 2
```

### 应用场景

**需要保存状态信息（0/1 即可表示）的场景**

- 举例：用户签到情况、活跃用户情况、用户行为统计（比如是否点赞过某个视频）。记录用户一年的签到情况。
- 相关命令：`SETBIT`、`GETBIT`、`BITCOUNT`、`BITOP`。

## HyperLogLog（基数统计）

HyperLogLog 是一种有名的**基数计数**概率算法 ，基于 LogLog Counting(LLC)优化改进得来，并不是 Redis 特有的，Redis 只是实现了这个算法并提供了一些开箱即用的 API。

Redis 提供的 HyperLogLog 占用空间非常非常小，只需要 12k 的空间就能存储接近`2^64`个不同元素。这是真的厉害，这就是数学的魅力么！并且，Redis 对 HyperLogLog 的存储结构做了优化，采用两种方式计数：

- **稀疏矩阵**：计数较少的时候，占用空间很小。
- **稠密矩阵**：计数达到某个阈值的时候，占用 12k 的空间。

基数计数概率算法为了节省内存并不会直接存储元数据，而是通过一定的概率统计方法预估基数值（集合中包含元素的个数）。因此， HyperLogLog 的计数结果并不是一个精确值，存在一定的误差（标准误差为 `0.81%` ）。

![img](pictures/image-20220720194154133.png)

HyperLogLog 的使用非常简单，但原理非常复杂。HyperLogLog 的原理以及在 Redis 中的实现可以看这篇文章：[HyperLogLog 算法的原理讲解以及 Redis 是如何应用它的open in new window](https://juejin.cn/post/6844903785744056333) 。

再推荐一个可以帮助理解 HyperLogLog 原理的工具：[Sketch of the Day: HyperLogLog — Cornerstone of a Big Data Infrastructureopen in new window](http://content.research.neustar.biz/blog/hll.html) 。

### 常用命令

| 命令                                      | 介绍                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| PFADD key element1 element2 ...           | 添加一个或多个元素到 HyperLogLog 中                          |
| PFCOUNT key1 key2                         | 获取一个或者多个 HyperLogLog 的唯一计数。                    |
| PFMERGE destkey sourcekey1 sourcekey2 ... | 将多个 HyperLogLog 合并到 destkey 中，destkey 会结合多个源，算出对应的唯一计数。 |

**基础操作**

```
> PFADD hll foo bar zap
(integer) 1
> PFADD hll zap zap zap
(integer) 0
> PFADD hll foo bar
(integer) 0
> PFCOUNT hll
(integer) 3
> PFADD some-other-hll 1 2 3
(integer) 1
> PFCOUNT hll some-other-hll
(integer) 6
> PFMERGE desthll hll some-other-hll
"OK"
> PFCOUNT desthll
(integer) 6
```

### 应用场景

**数量量巨大（百万、千万级别以上）的计数场景**

- 举例：热门网站每日/每周/每月访问 ip 数统计、热门帖子 uv 统计、
- 相关命令：`PFADD`、`PFCOUNT` 。

## Geospatial（地理位置）

Geospatial index（地理空间索引，简称 GEO） 主要用于**存储地理位置信息**，基于 Sorted Set 实现。

通过 GEO 我们可以轻松实现两个位置距离的计算、获取指定位置附近的元素等功能。

![img](pictures/image-20220720194359494.png)

### **常用命令**

| 命令                                             | 介绍                                                         |
| ------------------------------------------------ | ------------------------------------------------------------ |
| GEOADD key longitude1 latitude1 member1 ...      | 添加一个或多个元素对应的经纬度信息到 GEO 中                  |
| GEOPOS key member1 member2 ...                   | 返回给定元素的经纬度信息                                     |
| GEODIST key member1 member2 M/KM/FT/MI           | 返回两个给定元素之间的距离                                   |
| GEORADIUS key longitude latitude radius distance | 获取指定位置附近 distance 范围内的其他元素，支持 ASC(由近到远)、DESC（由远到近）、Count(数量) 等参数 |
| GEORADIUSBYMEMBER key member radius distance     | 类似于 GEORADIUS 命令，只是参照的中心点是 GEO 中的元素       |

**基础操作**

```
> GEOADD personLocation 116.33 39.89 user1 116.34 39.90 user2 116.35 39.88 user3
3
> GEOPOS personLocation user1
116.3299986720085144
39.89000061669732844
> GEODIST personLocation user1 user2 km
1.4018
```

GEO 中存储的地理位置信息的经纬度数据通过 GeoHash 算法转换成了一个整数，这个整数作为 Sorted Set 的 score(权重参数)使用。

![img](pictures/image-20220721201545147.png)

**获取指定位置范围内的其他元素**：

```
> GEORADIUS personLocation 116.33 39.87 3 km
user3
user1
> GEORADIUS personLocation 116.33 39.87 2 km
> GEORADIUS personLocation 116.33 39.87 5 km
user3
user1
user2
> GEORADIUSBYMEMBER personLocation user1 5 km
user3
user1
user2
> GEORADIUSBYMEMBER personLocation user1 2 km
user1
user2
```

`GEORADIUS` 命令的底层原理解析可以看看阿里的这篇文章：[Redis 到底是怎么实现“附近的人”这个功能的呢？](https://juejin.cn/post/6844903966061363207) 。

**移除元素**：

GEO 底层是 Sorted Set ，你可以对 GEO 使用 Sorted Set 相关的命令。

```
> ZREM personLocation user1
1
> ZRANGE personLocation 0 -1
user3
user2
> ZSCORE personLocation user2
4069879562983946
```

### 应用场景

**需要管理使用地理空间数据的场景**

- 举例：附近的人。
- 相关命令: `GEOADD`、`GEORADIUS`、`GEORADIUSBYMEMBER` 。

# 通用命令

| 命令                   | 介绍                                                         |
| ---------------------- | ------------------------------------------------------------ |
| KEYS *                 | 查询当去数据库下所有的key                                    |
| DEL key                | 删除指定key                                                  |
| UNLINK key             | 非阻塞删除，仅仅将key从keyspace元数据中删除，真正的删除会后续异步操作 |
| TYPE key               | 查看指定key的类型                                            |
| EXISTS key             | 查看指定key是否存在当前数据库中                              |
| TTL key                | 查看指定key多少秒过期，-1表示永不过期                        |
| EXPIRE key seconds     | 给指定key设置过期时间                                        |
| MOVE key dbindex[0-15] | 将当前数据库的key移动到指定数据库                            |
| SELECT dbindex[0-15]   | 选中数据库，默认为0                                          |
| DBSIZE                 | 查看当前数据库key的数量                                      |
| FLUSHDB                | 清除当前数据库中所有的内容                                   |
| FLUSHALL               | 清除当前实例中所有数据库中的内容                             |

# String 还是 Hash 存储对象数据更好呢？

- String 存储的是序列化后的对象数据，存放的是整个对象。Hash 是对对象的每个字段单独存储，可以获取部分字段的信息，也可以修改或者添加部分字段，节省网络流量。如果对象中某些字段需要经常变动或者经常需要单独查询对象中的个别字段信息，Hash 就非常适合。
- String 存储相对来说更加节省内存，缓存相同数量的对象数据，String 消耗的内存约是 Hash 的一半。并且，存储具有多层嵌套的对象时也方便很多。如果系统对性能和资源消耗非常敏感的话，String 就非常适合。

在绝大部分情况，我们建议使用 **String** 来存储对象数据即可

