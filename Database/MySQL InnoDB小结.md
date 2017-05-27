# MySQL InnoDB小结

## MySQL体系结构

MySQL特点：插件式体系结构。

![img](http://dl.iteye.com/upload/attachment/0064/2501/051d244c-d75d-372a-bc5a-6a14fe76a9ae.jpg)

由：

- 管理和服务组件
- 连接池组件
- sql接口组件
- 查询分析器组件
- 优化器组件
- 缓冲组件
- 插件式存储引擎
- 物理文件

组成。

## MySQL各个存储引擎

- InnoDB

  面向OLTP（Online Transaction Processing，在线事务处理）、行锁、支持外键、非锁定读、默认采用repeaable（可重复读）级别、通过next-key locking策略避免幻读、插入缓冲、二次写、自适应哈希索引、预读

- MyISAM

  不支持事务、表锁、全文索引、适合OLAP（Online  Analysis Processing，在线分析处理）。其中myd:放数据文件，myi:放索引文件 

- ndb

  集群存储引擎，share nothing，可提高可用性

- memory

  数据存放在内存中，表锁，并发性能差，默认使用哈希索引

- archive

  只支持insert和select zlib算法压缩1：10，适合存储归档数据如日志等、行锁

- maria

  目的取代myisam、缓存数据和索引、行锁、mvcc


![img](http://dl.iteye.com/upload/attachment/0064/2503/2b28ecc9-a276-3182-bbc6-a2f6e8e58a97.jpg)





## InnoDB特性

### 主体系结构

默认7个后台线程。

4个IO Thread、

- insert buffer
- log
- read
- write

1个Master Thread（优先级最高）、

1个锁（lock）监控线程、

1个错误监控线程，可以通过 `SHOW ENGINE innodb STATUS` 来查看。

新版本已对默认的read thread和write thread增大到4个，可以通过 `SHOW VARIABLES LIKE 'innodb_io_thread%'` 查看。

### 存储引擎组成

- 缓冲池（buffer pool）
- 重做日志缓冲（redo log buffer）
- 额外的内存池（additional memory pool）

![img](http://dl.iteye.com/upload/attachment/0064/2505/7a6f60a9-bc4a-30e5-ba13-859e65c7e3de.jpg)



### 缓冲池

最大块内存，用来存放各种数据的缓存包括：

- 有索引页
- 数据页
- undo页
- 插入缓冲
- 自适应哈希索引
- innodb存储的锁信息
- 数据字典信息等。

工作方式总是将数据库文件按页（每页16K）读取到缓冲池，然后按最近最少使用（LRU）的算法来保留在缓冲池中的缓存数据。如果数据库文件需要修改，总是首先修改在缓存池中的页（发生修改后即为脏页），然后再按照一定的频率将缓冲池的脏页刷新到文件。

### 日志缓冲

将重做日志先放入这个缓冲区，然后按照一定的频率刷新到重做日志文件。

### Master Thread

主循环（loop）执行每秒一次的操作：

- 日志缓冲刷新到磁盘，即使这个事务还没有提交（总是执行，所以再大的事务commit的时间也是很快的）
- 合并插入缓冲（innodb当前一秒发生的io次数小于5次则执行）
- 至多刷新100个innodb的缓冲池中的脏页到磁盘（超过配置的脏页所占缓冲池比例则执行，在配置文件中innodb_max_dirty_pages_pac决定，默认是90，新版本是75，google建议是80）
- 如果当前没用用户活动，切换到backgroud loop



http://blog.csdn.net/zhengfeng2100/article/details/51850493

   