### MySQL的三种日志

#### redo log

重做日志

+ 作用：确保事务的持久性。防止在发生故障的时间点，尚有脏页未写入磁盘，在重启MySQL服务的时候，根据redo log进行重做，从而达到事务的持久性这一特性。

+ 内容：物理格式的日志，记录的是物理数据页面的修改的信息，其redo log是顺序写入redo log file的物理文件中去的。在事务提交时，必须将改事务的所有日志写入到重做日志文件进行持久化。

> MySQL，如果每次更新操作都要写进磁盘，然后磁盘要找到对应记录，然后再更细，整个过程io成本、查找成本都很高。
>
> 解决方案：WAL技术（Write-Ahead Logging）。先写日志，再写磁盘。
>
> 具体来说，当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。
>
> ![redo log](https://github.com/codzeroNov/MyNotes/blob/master/MySQL/PICS/redo%20log.png)
>
> write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。
>
> write pos 和 checkpoint 之间的是log上还空着的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，表示log满了，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。
>
> 有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。

#### binlog

归档日志（二进制日志）

+ 作用：1.用于复制，在主从复制中，从库利用主库上的binlog进行重播，实现主从同步。2. 用于数据库的基于时间点 ( point - in - time) 的还原。

+ 内容：逻辑格式的日志，可以简单认为就是执行过的事务中的sql语句。

> 但又不完全是sql语句这么简单，而是包括了执行的sql语句（增删改）反向的信息，也就意味着delete对应着delete本身和其反向的insert；update对应着update执行前后的版本的信息；insert对应着delete和insert本身的信息。

+ binlog 有三种模式：Statement（基于 SQL 语句的复制）、Row（基于行的复制） 以及 Mixed（混合模式）

　　MySQL 整体来看，其实就有两块：一块是 Server 层，它主要做的是 MySQL 功能层面的事情；还有一块是引擎层，负责存储相关的具体事宜。redo log 是 InnoDB 引擎特有的日志，而 Server 层也有自己的日志，称为 binlog（归档日志）。

　　这两种日志有以下三点不同：

1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

> **两阶段提交（2PC）保证两个日志的一致性**
>
> update 语句时的内部流程：
>
> 1. 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
>
> 2. 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
>
> 3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。
>
> 4.  执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
>
> 5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。
>
> ![binlog redolog 2pc](https://github.com/codzeroNov/MyNotes/blob/master/MySQL/PICS/binlog%20redo%202pc.png)

#### undo log

回滚日志

+ 作用：保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读

+ 内容：逻辑格式的日志，在执行undo的时候，仅仅是将数据从逻辑上恢复至事务之前的状态，而不是从物理页面上操作实现的，这一点是不同于redo log的。

#### 三种日志的区别

+ **redo log**在事务没有提交前，每一个修改操作都会记录变更后的数据，保存的是物理日志->数据。
  防止在发生故障的时间点，尚有脏页未写入磁盘，在重启mysql服务的时候，根据redo log进行重做，从而达到事务的持久性。
  redo log只是先写入Innodb_log_buffer，定时fsync到磁盘。

+ **binlog**只会在日志提交后，一次性记录执行过的事务中的sql语句以及其反向sql(作为回滚用)，保存的是逻辑日志->执行的sql语句。
+ **undo log**事务开始之前，将当前版本生成undo log，undo 也会产生 redo 来保证undo log的可靠性，保存的是逻辑日志->数据前一个版本。

**执行效率**

1. 基于redo log直接恢复数据的效率高于基于binlog sql语句恢复。
2. binlog不是循环使用，在写满或者重启之后，会生成新的binlog文件，redo log是循环使用。
3. binlog可以作为恢复数据使用，主从复制搭建，redo log作为异常宕机或者介质故障后的数据恢复使用。

---

### InnoDB基于undo log实现MVCC

>  MySQL数据库中读分为一致性非锁定读(Consistent Nonlocking Reads)、一致性锁定读(Consistent Locking Reads)
> 一致性非锁定读（快照读），普通的SELECT，通过多版本并发控制（MVCC -- Multi-Version Concurrency Control）实现。
> 一致性锁定读（当前读），SELECT ... FOR UPDATE;/SELECT ... LOCK IN SHARE MODE;/INSERT...;/UPDATE...;/DELETE...，通过锁实现。

**读提交**

在事务T中，**每次读**都会创建一个视图read-view。

**可重复读**

事务T启动后**第一次读**的时候会创建一个视图 read-view，之后事务 T 执行期间，即使有其他事务修改了数据，事务 T 看到的仍然跟在启动时看到的一样。文章下面基于可重复读进行讲解。

#### 隐藏列

在分析MVCC原理之前，先看下InnoDB中数据行的结构：

![structure of rows in innodb](https://github.com/codzeroNov/MyNotes/blob/master/MySQL/PICS/structure%20of%20rows%20in%20innodb.png)

在InnoDB中，每一行都有2个隐藏列DATA_TRX_ID和DATA_ROLL_PTR(如果没有定义主键，则还有个隐藏主键列)：

1. DATA_TRX_ID表示最近修改该行数据的事务ID

2. DATA_ROLL_PTR则表示指向该行回滚段的指针，该行上所有旧的版本，在undo中都通过链表的形式组织，而该值，正式指向undo中该行的历史记录链表

整个MVCC的关键就是通过DATA_TRX_ID和DATA_ROLL_PTR这两个隐藏列来实现的。

#### 版本链

undo log是为回滚而用，具体内容就是copy事务前的数据库内容（行）到undo buffer，在适合的时间把undo buffer中的内容刷新到磁盘。undo buffer与redo buffer一样，也是环形缓冲，但当缓冲满的时候，undo buffer中的内容会也会被刷新到磁盘；与redo log不同的是，磁盘上不存在单独的undo log文件，所有的undo log均存放在主ibd数据文件中（表空间），即使客户端设置了每表一个数据文件也是如此。

undo log 在 Rollback segment中又被细分为 insert 和 update undo log ，insert 类型的undo log 仅仅用于事务回滚，当事务一旦提交，insert undo log 就会被丢弃。update的undo log 被用于一致性读和事务回滚，update undo log 的清理是在没有事务需要对这部分数据快照进行一致性读的时候进行清理。

undo log 的创建
每次对数据进行更新操作时，都会copy当前数据，保存到undo log 中。并修改当前行的回滚指针指向undo log中的旧数据行。

![version list of innodb](https://github.com/codzeroNov/MyNotes/blob/master/MySQL/PICS/version%20list%20of%20innodb.webp)

#### 事务匹配原理

在执行查询SQL时，会生成一致性视图read-view，InnoDB 为每个事务构造了一个数组，用来保存这个事务启动瞬间，当前正处于启动了但还没提交的所有事务ID。数组里面事务 ID 的最小值记为低水位（min_id），当前系统里面已经创建过的事务 ID 的最大值加1记为高水位（max_id）。

![watermarks of transation](D:\DOCS\PICS\watermarks of transation.png)

这个视图数组和高水位，就组成了当前事务的一致性视图（read-view），而数据版本的可见性规则，就是基于数据行trx_id 和这个一致性视图的对比结果得到的。这个视图数组把所有的行的trx_id 分成了几种不同的情况:

1. 若 trx_id < min_id，表示这个版本是已提交事务生成的，当前版本可见。
2. 若 trx_id >max_id，表示这个版本是由将来启动的事务生成，当前版本不可见。
3. 若 min_id <= trx_id <= max_id，则包含两种情况：
   + 若行的trx_id在数组中，则表示该版本由未提交事务生成，当前事务不可见。
   + 若行的trx_id不在数组中，则表示该版本由已提交的事务生成，当前事务可见。

> 删除的情况可以看作update的特殊情况处理。删除时，会将版本链上最新的数据赋值一份，然后将trx_id修改成删除的trx_id，同时将该条记录的头信息（record_header）里的删除标记（delete_flag）上标为true，来表示当前记录已被删除，在查询时在版本链上遍历时，找到版本为已提交和删除标记为true的记录时则不返回数据。

#### 例子1

前置条件：表mvcc_test中初始为空。含id, book_name, price三个字段。

|      |                     事务T1                      |                    事务T2                    |               事务T3                |
| :--: | :---------------------------------------------: | :------------------------------------------: | :---------------------------------: |
|  1   |                     begin;                      |                    begin;                    |               begin;                |
|  2   |                                                 | INSERT INTO mvcc_test VALUE(1, "书A", 29.8); |                                     |
|  3   |                                                 |                   COMMIT;                    |                                     |
|  4   |                                                 |                                              | SELECT FROM mvcc_test WHERE id = 1; |
|  5   | UPDATE mvcc_test SET price = 26.8 WHERE id = 1; |                                              |                                     |
|  6   |                                                 |                                              | SELECT FROM mvcc_test WHERE id = 1; |
|  7   |                     COMMIT;                     |                                              |               COMMIT;               |

Q1：事务T3在时间点4是否能读到表mvcc_test中的数据？

Q2：事务T3在时间点6读到书的价格为？



> A1：在RR隔离级别下，快照并不是在BEGIN就开始产生了，而是要等到事务当中的第一次查询之后才会产生快照，之后的查询就只读取这个快照数据。所以事务T3第4行读得到数据，书的价格为29.8。
>
> A2： 因为快照在第一个查询时生成，所以即使T1在第5行更改了书的价格，T3读到的价格仍为29.8。

#### 例子2（进阶）

表mvcc_test中存在(1, "书A", 26.8)。

|      |                    事务T1                     |                    事务T2                     |               事务T3                |
| :--: | :-------------------------------------------: | :-------------------------------------------: | :---------------------------------: |
|  1   |                    begin;                     |                    begin;                     |               begin;                |
|  2   | UPDATE mvcc_test SET price = 28 WHERE id = 1; |                                               |                                     |
|  3   |                                               | UPDATE mvcc_test SET price = 23 WHERE id = 1; |                                     |
|  4   |                                               |                    COMMIT;                    |                                     |
|  5   |                                               |                                               | SELECT FROM mvcc_test WHERE id = 1; |
|  6   |                    COMMIT;                    |                                               |                                     |
|  7   |                                               |                                               | SELECT FROM mvcc_test WHERE id = 1; |
|  8   |                                               |                                               |               COMMIT;               |

Q1：事务T3在时间点5读到书的价格为？

Q2：事务T3在时间点7读到书的价格为？



> A1：在时间点5时活跃的事务为T1，T3（T2已提交），所以数组的min_id为T1，max_id为T4。从id = 1的数据行的回滚指针，往前遍历到trx_id = 2的记录，发现min_id < trx_id < max_id，属于情况3，又因T2不在活跃数组中，属于子情况2，对当前事务可见，所以T3读到的价格为23。
>
> A2： 因为快照在第一个查询时生成，所以即使T1在第5行更改了书的价格，T3读到的价格仍为23。

