# Redis

## 1. 什么是Redis？

Redis是一个开源的、基于内存的、高性能key-value型数据库。

## 2. Redis的特点

-   Redis是一个内存型数据库，因此性能十分出色；
-   Redis支持数据的持久化，通过异步操作将内存中的数据存储到磁盘中；
-   Redis支持多种数据结构，不仅支持传统的key-value型数据，还提供list、set、zset、hash等数据结构；
-   Redis的单个value容量最大限制是1GB，同时可以记录数据的最后访问时间，因此可以实现很多有用的功能。
-   Redis的主要缺点是数据库容量收到物理内存的限制，不能用作海量数据的高性能读写，因此Redis的使用场景主要局限在较少量数据的高性能操作和运算上。

## 3. 使用Redis有哪些好处？

-   速度快，因为数据存在内存中，类似于HashMap，HashMap的优势就是查找和操作的时间复杂度都是O(1)；
-   支持丰富数据类型，支持string，list，set，sorted set，hash；
-   支持事务，操作都是原子性，所谓的原子性就是对数据的更改要么全部执行，要么全部不执行；
-   丰富的特性：可用于缓存、消息，按key设置过期时间，过期后将会自动删除。

## 4. Redis相比memcached有哪些优势？

-   memcached所有的值均是简单的字符串，redis作为其替代者，支持更为丰富的数据类型；
-   redis的速度比memcached快很多；
-   redis可以持久化其数据。

## 5. Redis支持的数据类型

Redis用到的主要数据结构有：简单动态字符串（SDS）、双端链表（list）、字典（dict）、跳跃表（skiplist）、压缩列表（ziplist）、整数集合（intset）等等。

Redis并没有直接使用上述的数据结构来实现，而是基于这些数据结构创建了一个对象系统，包括字符串对象（REDIS_STRING)、列表对象（REDIS_LIST）、哈希对象（rEDIS_HASH）、集合对象（REDIS_SET）和有序集合对象（REDIS_ZSET）。

在Redis保存的键值对（key-value）中，键总是一个字符串对象，而值可以是上述对象的任何一种。

## 6. 为什么Redis要把所有数据放到内存中？

Redis为了达到最快的读写速度将数据都读到内存中，并通过异步的方式将数据写入磁盘。所以redis具有快速和数据持久化的特征。如果不将数据放在内存中，磁盘I/O速度为严重影响redis的性能。在内存越来越便宜的今天，redis将会越来越受欢迎。

如果设置了最大使用的内存，则数据已有记录数达到内存限值后不能继续插入新值。

## 7. Redis是单进程单线程的吗？

是的。Redis利用队列技术将并发访问变为串行访问，消除了传统数据库串行控制的开销。

## 8. 虚拟内存

当你的key很小而value很大时，使用VM的效果会比较好。因为这样节约的内存比较大。

当你的key不小时，可以考虑使用一些非常方法将很大的key变成很大的value，比如你可以考虑将key-value组合成一个新的value。

修改vm-max-threads这个参数，可以设置访问swap文件的线程数，设置最好不要超过机器的核数，如果设置为0，那么所有对swap文件的操作都是串行的，可能会造成比较长时间的延迟，但是对数据完整性有很好的保证。

如果数据量很大，可以考虑分布式或者其他数据库。

## 9. 分布式

Redis支持主从的模式。原则：master会将数据同步到slave，而slave不会将数据同步到master。slave启动时会连接master来同步数据。

这是一个典型的分布式读写分离模型。我们可以利用master来插入数据，slave提供检索服务。这样可以有效减少单个机器的并发访问数量。

## 10. 读写分离模型

通过增加slave DB的数量，读的性能可以线性增长。为了避免master DB的单点故障，集群一般都会采用两台master DB做双机热备，所以整个集群的读和写的可用性都非常高。

读写分离[架构](http://lib.csdn.net/base/architecture)的缺陷在于，不管是Master还是Slave，每个节点都必须保存完整的数据，如果在数据量很大的情况下，集群的扩展能力还是受限于单个节点的存储能力，而且对于Write-intensive类型的应用，读写分离架构并不适合。

## 11. 数据分片模型

为了解决读写分离模型的缺陷，可以将数据分片模型应用进来。可以将每个节点看成都是独立的master，然后通过业务实现数据分片。结合上面两种模型，可以将每个master设计成由一个master和多个slave组成的模型。

## 12. Redis的回收策略

-   volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰；
-   volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰；
-   volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰；
-   allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰；
-   allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰；
-   no-enviction（驱逐）：禁止驱逐数据。

## 13. Redis常见性能问题和解决方案

-   Master写内存快照，save命令调度rdbSave函数，会阻塞主线程的工作，当快照比较大时对性能影响是非常大的，会间断性暂停服务，所以Master最好不要写内存快照；
-   Master AOF持久化，如果不重写AOF文件，这个持久化方式对性能的影响是最小的，但是AOF文件会不断增大，AOF文件过大会影响Master重启的恢复速度。Master最好不要做任何持久化工作，包括内存快照和AOF日志文件，特别是不要启用内存快照做持久化,如果数据比较关键，某个Slave开启AOF备份数据，策略为每秒同步一次；
-   Master调用BGREWRITEAOF重写AOF文件，AOF在重写的时候会占大量的CPU和内存资源，导致服务load过高，出现短暂服务暂停现象；
-   Redis主从复制的性能问题，为了主从复制的速度和连接的稳定性，Slave和Master最好在同一个局域网内。

## 14. MySQL里有2000W数据，Redis中只存20W的数据，如何保证Redis中的数据都是热点数据？

Redis内存数据集大小上升到一定大小的时候，就会施行数据淘汰策略。见“12. Redis的回收策略”。

## 15. Redis最适合的场景

Redis最适合所有数据in-momory的场景，虽然Redis也提供持久化功能，但实际更多的是一个disk-backed的功能，跟传统意义上的持久化有比较大的差别，那么可能大家就会有疑问，似乎Redis更像一个加强版的Memcached，那么何时使用Memcached,何时使用Redis呢?

 如果简单地比较Redis与Memcached的区别，大多数都会得到以下观点：

1.  Redis不仅仅支持简单的k/v类型的数据，同时还提供list，set，zset，hash等[数据结构](http://lib.csdn.net/base/datastructure)的存储。
2.  Redis支持数据的备份，即master-slave模式的数据备份。
3.  Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用。

### 会话缓存（Session Cache）

最常用的一种使用Redis的情景是会话缓存（session cache）。用Redis缓存会话比其他存储（如Memcached）的优势在于：Redis提供持久化。当维护一个不是严格要求一致性的缓存时，如果用户的购物车信息全部丢失，大部分人都会不高兴的，现在，他们还会这样吗？

幸运的是，随着 Redis 这些年的改进，很容易找到怎么恰当的使用Redis来缓存会话的文档。甚至广为人知的商业平台Magento也提供Redis的插件。

### 全页缓存（FPC）

除基本的会话token之外，Redis还提供很简便的FPC平台。回到一致性问题，即使重启了Redis实例，因为有磁盘的持久化，用户也不会看到页面加载速度的下降，这是一个极大改进，类似PHP本地FPC。

再次以Magento为例，Magento提供一个插件来使用Redis作为[全页缓存后端](https://github.com/colinmollenhour/Cm_Cache_Backend_Redis)。

此外，对WordPress的用户来说，Pantheon有一个非常好的插件  [wp-redis](https://wordpress.org/plugins/wp-redis/)，这个插件能帮助你以最快速度加载你曾浏览过的页面。

### 队列

Reids在内存存储引擎领域的一大优点是提供 list 和 set 操作，这使得Redis能作为一个很好的消息队列平台来使用。Redis作为队列使用的操作，就类似于本地程序语言（如[Python](http://lib.csdn.net/base/python)）对 list 的 push/pop 操作。

如果你快速的在Google中搜索“Redis queues”，你马上就能找到大量的开源项目，这些项目的目的就是利用Redis创建非常好的后端工具，以满足各种队列需求。例如，Celery有一个后台就是使用Redis作为broker，你可以从[这里](http://celery.readthedocs.org/en/latest/getting-started/brokers/redis.html)去查看。

### 排行榜/计数器

Redis在内存中对数字进行递增或递减的操作实现的非常好。集合（Set）和有序集合（Sorted Set）也使得我们在执行这些操作的时候变的非常简单，Redis只是正好提供了这两种数据结构。所以，我们要从排序集合中获取到排名最靠前的10个用户–我们称之为“user_scores”，我们只需要像下面一样执行即可：

当然，这是假定你是根据你用户的分数做递增的排序。如果你想返回用户及用户的分数，你需要这样执行：

ZRANGE user_scores 0 10 WITHSCORES

Agora Games就是一个很好的例子，用Ruby实现的，它的排行榜就是使用Redis来存储数据的，你可以在[这里]()看到。

### 发布/订阅

最后（但肯定不是最不重要的）是Redis的发布/订阅功能。发布/订阅的使用场景确实非常多。我已看见人们在社交网络连接中使用，还可作为基于发布/订阅的脚本触发器，甚至用Redis的发布/订阅功能来建立聊天系统！（不，这是真的，你可以去核实）。

Redis提供的所有特性中，我感觉这个是喜欢的人最少的一个，虽然它为用户提供如果此多功能。

