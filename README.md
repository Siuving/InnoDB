# InnoDB



## 第二章 InonoDB 存储引擎



### 2.3 InnoDB 体系架构

InnoDB 存储引擎有多个内存块，可以认为这些内存块组成了一个大的内存池，负责如下工作：

*   维护所有进程/线程需要访问的多个内部数据结构。
*   缓存磁盘上的数据，方便快速地读取，同时在对磁盘文件的数据修改之前在这里缓存。
*   重做日志（redo log）缓冲。

...

![](./pictures/InnoDB存储架构.jpg)

<div align = "center"><strong>图 2-1 &nbsp InnoDB 存储引擎体系架构</strong></div>

**后台线程** 的主要作用是负责刷新内存池中的数据，保证缓冲池中的内存缓存的是最近的数据。此外将已修改的数据文件刷新到磁盘文件，同时保证在数据库发生异常的情况下 InnoDB 能恢复到正常运行状态。



#### 2.3.1 后台线程

InnoDB 存储引擎是 **多线程** 的模型，因此其后台有多个不同的后台线程，负责处理不同的任务。



**1) Master Thread**

Master Thread 是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插人缓冲 (INSERT BUFFER)、UNDO 页的回收等。



**2) IO Thread**

在 InnoDB 存储引擎中大量使用了 AIO (Async IO）来处理写 IO 请求，这样可以极大提高数据库的性能。而 IO Thread 的工作主要是负责这些 IO 请求的 **回调(call back)处理**。

```mysql
show variables like 'innodb_version'\G;		-- InnoDB 版本

/*

*************************** 1. row ***************************
Variable_name: innodb_version
        Value: 5.7.29
1 row in set (0.02 sec)

*/
```

观察 InnoDB 中的 IO Threads：

```mysql
show engine innodb status\G;

/*

*************************** 1. row ***************************
  Type: InnoDB
  Name: 
Status: 
=====================================
2023-04-12 20:38:15 0x7f79f01cb700 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 21 seconds
-----------------
...
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (log thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (read thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
I/O thread 9 state: waiting for completed aio requests (write thread)
...
----------------------------
END OF INNODB MONITOR OUTPUT
============================

1 row in set (0.00 sec)


*/
```

可以看到 IO Thread 0 为 **insert buffer thread**。IO Thread 1 为 **log thread**。之后就是根据参数innodb_read_io_threads 及 innodb_write_io_threads 来设置的读写线程，并且读线程的 ID 总是小于写线程。



**3) Purge Thread**

事务被提交后，其所使用的 undolog 可能不再需要，因此需要 PurgeThread 来回收已经使用并分配的 undo 页。

```mysql
show variables like 'innodb_purge_threads'\G;

/*

*************************** 1. row ***************************
Variable_name: innodb_purge_threads
        Value: 4
1 row in set (0.00 sec)


从 InnoDB 1.2 版本开始，InnoDB 支持多个 Purge Thread，这样做的目的是为了进。步加快 undo 页的回收。同时由于 Purge Thread 需要离散地读取 undo 页，这样也能更进一步利用磁盘的随机读取性能。如用户可以设置 4 个 Purge Thread.


*/
```



**4) Page Cleaner Thread**

Page Cleaner Thread 是在 InnoDB 1.2.x 版本中引入的。其作用是将之前版本中脏页的刷新操作都放入到单独的线程中来完成。而其目的是为了减轻原 Master Thread 的工作及对于用户查询线程的阻塞，进一步提高 InnoDB 存储引擎的性能。



#### 2.3.2 内存



**1)  缓冲池**

InnoDB 存储引擎是 **基于磁盘存储** 的，并将其中的记录按照页的方式进行管理。因此可将其视为基于磁盘的数据库系统（Disk-base Database)。在数据库系统中，由于CPU速度与磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池技术来提高数据库的整体性能。

缓冲池简单来说就是一块 **内存区域**，通过内存的速度来弥补磁盘速度较慢对数据库性能的影响。在数据库中进行读取页的操作，首先将从磁盘读到的页存放在缓冲池中，这个过程称为将页 “FIX” 在缓冲池中。下一次再读相同的页时，首先判断该页是否在缓冲池中。若在缓冲池中，称该页在缓冲池中被命中，直接读取该页。否则，读取磁盘上的页。

对于数据库中页的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。这里需要注意的是，页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时触发，而是通过一种称为 Checkpoint 的机制刷新回磁盘。同样，这也是为了提高数据库的整体性能。



缓冲池配置：

```mysql
show variables like 'innodb_buffer_pool_size'\G;

/*

*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_size
        Value: 134217728
1 row in set (0.01 sec)

134217728 / 1024 / 1024 = 128 M 大小

*/
```



具体来看，缓冲池中缓存的数据页类型有：**索引页**、**数据页**、**undo 页**、**插入缓冲(insert buffer)**、**自适应哈希索引(adaptive hash index)**、**InnoDB 存储的锁信息（lockinfo)**、**数据字典信息（data dictionary）**等。不能简单地认为，缓冲池只是缓存索引页和数据页，它们只是占缓冲池很大的一部分而已。

![](./pictures/InnoDB内存数据对象.jpg)

<div align = "center"><strong>图 2-2 &nbsp InnoDB 内存数据对象</strong></div>

查看缓冲池实例个数：

```mysql
show variables like 'innodb_buffer_pool_instances'\G;

/*

*************************** 1. row ***************************
Variable_name: innodb_buffer_pool_instances
        Value: 1
1 row in set (0.00 sec)

默认的缓冲池实例的个数

*/
```



查看缓冲池使用状态：

```mysql
use information_schema;
select pool_id, pool_size, free_buffers, database_pages 
from innodb_buffer_pool_stats\G;

/*

*************************** 1. row ***************************
       pool_id: 0
     pool_size: 8191
  free_buffers: 7935
database_pages: 256
1 row in set (0.00 sec)


*/
```



**2) LRU List、Free List 和 Flush List** 

通常来说，数据库中的缓冲池是通过 **LRU** ( Latest Recent Used，最近最少使用）算法来进行管理的。即最频繁使用的页在 LRU 列表的前端，而最少使用的页在 LRU 列表的尾端。当缓冲池不能存放新读取到的页时，将首先释放 LRU 列表中尾端的页。

在 InnoDB 存储引擎中，缓冲池中页的大小默认为16KB，同样使用 LRU 算法（优化版）对缓冲池进行管理。

InnoDB 的存储引擎中，LRU 列表中还加入了 midpoint 位置。新读取到的页，虽然是最新访问的页，但并不是直接放入到LRU 列表的首部，而是放入到 LRU 列表的 midpoint 位置。

```mysql
show variables like 'innodb_old_blocks_pct'\G;

/*

*************************** 1. row ***************************
Variable_name: innodb_old_blocks_pct
        Value: 37				
1 row in set (0.00 sec)


*/
```

从上面的例子可以看到，参数 innodb_old_blocks _pct 默认值为 37，表示新读取的页插人到 LRU 列表尾端的 37% 的位置（差不多 3/8 的位置)。在 InnoDB 存储引擎中，把 midpoint 之后的列表称为 old 列表，之前的列表称为 new 列表。可以简单地理解为 new 列表中的页都是最为活跃的热点数据。

若直接将读取到的页放入到 LRU 的首部，那么某些 SQL 操作可能会使缓冲池中的页被刷新出，从而影响缓冲池的效率。

常见的这类操作为索引或数据的扫描操作。这类操作需要访问表中的许多页，甚至是全部的页，而这些页通常来说又仅在这次查询操作中需要，并不是活跃的热点数据。如果页被放入 LRU 列表的首部，那么非常可能将所需要的热点数据页从LRU 列表中移除，而在下一次需要读取该页时，InnoDB 存储引擎需要再次访问磁盘。



为了解决这个问题，InnoDB 存储引擎引入了另一个参数来进一步管理 LRU 列表，这个参数是 **innodb_old_blocks_time**，用于表示页读取到 mid 位置后需要等待多久才会被加人到 LRU 列表的热端。因此当需要执行上述所说的 SQL 操作时，可以通过下面的方法尽可能使 LRU 列表中热点数据不被刷出。

```mysql
show variables like 'innodb_old_blocks_time'\G;

/*

*************************** 1. row ***************************
Variable_name: innodb_old_blocks_time
        Value: 1000							默认是 1000 s
1 row in set (0.00 sec)

*/


-- set global innodb_old_blocks_time=1000;		设置等待时间
-- set global innodb_old_blocks_pct=20;			减少热点页可能刷出的概率
/*

改成 20 也就是新的页被插入到尾端 20% 的位置，前面的 80% 页可能安全

*/
```

注意︰执行命令 SHOW ENGINE INNODB STATUS 显示的不是当前的状态，而是过去某个时间范围内 InnoDB 存储引擎的状态。



LRU 列表用来管理已经读取的页，但当数据库刚启动时，LRU 列表是空的，即没有任何的页。这时页都存放在Free 列表中。当需要从缓冲池中分页时，首先从 Free 列表中查找是否有可用的空闲页，若有则将该页从 Free 列表中删除，放人到 LRU 列表中。

```mysql
/*
...
Buffer pool size   8191					总共的页数
Free buffers       7935					当前 Free 列表中页的个数
Database pages     256					LRU 列表中页的数量
Old database pages 0
Modified db pages  0					脏页数量
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0			 pages made young -> LRU 列表中页移动到前端的次数
0.00 youngs/s, 0.00 non-youngs/s
Pages read 221, created 35, written 39
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 256, unzip_LRU len: 0			压缩页

...


可能的情况是 Free buffers 与 Database pages 的数量之和不等于 Buffer pool size。因为缓冲池中的页还可能会被分配给自适应哈希索引、Lock 信息、Insert Buffer 等页，而这部分页不需要 LRU 算法进行维护，因此不存在于 LRU 列表中。


*/
```



查看缓冲池运行状态：

```mysql
select pool_id, hit_rate, pages_made_young, pages_not_made_young 
from information_schema.innodb_buffer_pool_stats\G;

/*

*************************** 1. row ***************************
             pool_id: 0
            hit_rate: 0
    pages_made_young: 0
pages_not_made_young: 0
1 row in set (0.00 sec)

可以多次去测试

*/
```



查看每个 LRU 列表中每个页的具体信息：

```mysql
use information_schema;
select table_name, space, page_number, page_type
from innodb_buffer_page_lru 
where space=1;

/*

Empty set (0.00 sec)		没查到啥..

*/

```



在 LRU 列表中的页被修改后，称该页为 **脏页**（dirty page)，即缓冲池中的页和磁盘上的页的数据产生了不一致。这时数据库会通过 CHECKPOINT 机制将脏页刷新回磁盘，而 **Flush 列表** 中的页即为脏页列表。需要注意的是，脏页既存在于 LRU 列表中，也存在于 Flush 列表中。LRU 列表用来管理缓冲池中页的可用性，Flush 列表用来管理将页刷新回磁盘，二者互不影响。

脏页同样存在于 LRU 列表中，修改查找条件即可：

```mysql
select table_name, space, page_number, page_type
from innodb_buffer_page_lru 
where oldest_modification > 0;

/*

Empty set (0.00 sec)		啥也没有..

*/
```



**3) 重做日志文件**

从图 2-2 可以看到，InnoDB 存储引擎的内存区域除了有缓冲池外，还有 **重做日志缓冲(redo log buffer)**。InnoDB 存储引擎首先将重做日志信息先放入到这个缓冲区，然后按一定频率将其刷新到重做日志文件。重做日志缓冲一般不需要设置得很大，因为一般情况下每一秒钟会将重做日志缓冲刷新到日志文件，因此用户只需要保证每秒产生的事务量在这个缓冲大小之内即可。

```mysql
show variables like 'innodb_log_buffer_size'\G;

/*

*************************** 1. row ***************************
Variable_name: innodb_log_buffer_size
        Value: 16777216							这里默认是 16 MB
1 row in set (0.00 sec)

*/
```

重做日志在下列三种情况下会将重做日志缓冲中的内容刷新到外部磁盘的重做日志文件中：

*   Master Thread 每一秒将重做日志缓冲刷新到重做日志文件;
*   每个事务提交时会将重做日志缓冲刷新到重做日志文件;
*   当重做日志缓冲池剩余空间小于 1/2 时，重做日志缓冲刷新到重做日志文件。



**4) 额外的内存池**

额外的内存池通常被 DBA 忽略，他们认为该值并不十分重要，事实恰恰相反，该值同样十分重要。在 InnoDB 存储引擎中，对内存的管理是通过一种称为 **内存堆（heap)** 的方式进行的。在对一些数据结构本身的内存进行分配时，需要从额外的内存池中进行申请，当该区域的内存不够时，会从缓冲池中进行申请。例如，分配了缓冲池（innodb_buffer_pool)，但是每个缓冲池中的帧缓冲(frame buffer）还有对应的缓冲控制对象(buffer control block)，这些对象记录了一些诸如 LRU、锁、等待等信息，而这个对象的内存需要从额外内存池中申请。因此，在申请了很大的 InnoDB 缓冲池时，也应考虑相应地增加这个值。



### 2.4 Checkpoint 技术



