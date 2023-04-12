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



**1.Master Thread**

Master Thread 是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新、合并插人缓冲 (INSERT BUFFER)、UNDO 页的回收等。



**2.IO Thread**

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



