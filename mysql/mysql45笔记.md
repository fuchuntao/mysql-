# mysql笔记

## 1、基础架构

### 1.1、问题：

1. MySQL的框架有几个组件, 各是什么作用? 
2. Server层和存储引擎层各是什么作用?
3. you have an error in your SQL syntax 这个保存是在词法分析里还是在语法分析里报错?
4. 对于表的操作权限验证在哪里进行?
5. 执行器的执行查询语句的流程是什么样的?
6. 如果表 T 中没有字段 k，而你执行了这个语句 select * from T where k=1, 那肯定是会报“不存在这个列”的错误： “Unknown column ‘k’ in ‘where clause’”。你觉得这个错误是在我们上面提到的哪个阶段报出来的呢？



### 1.2、内容：



#### 1.2.1、mysql的基本架构图

MySQL 可以分为 Server 层和存储引擎层两部分

![img](https://static001.geekbang.org/resource/image/0d/d9/0d2070e8f84c4801adbfa03bda1f98d9.png)



#### 1.2.2、连接器

```sql
--远程连接数据库命令
mysql -h$ip -P$port -u$user -p

--查看连接状态（command列显示）
show processlist;
```

客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数 wait_timeout 控制的，默认值是 8 小时。

但是全部使用长连接后，你可能会发现，有些时候 MySQL 占用内存涨得特别快，这是因为 MySQL 在执行过程中临时使用的内存是管理在连接对象里面的。这些资源会在连接断开的时候才释放。所以如果长连接累积下来，可能导致内存占用太大，被系统强行杀掉（OOM），从现象看就是 MySQL 异常重启了。

怎么解决这个问题呢？你可以考虑以下两种方案。

1. 定期断开长连接。使用一段时间，或者程序里面判断执行过一个占用内存的大查询后，断开连接，之后要查询再重连。
2. 如果你用的是 MySQL 5.7 或更新版本，可以在每次执行一个比较大的操作后，通过执行 mysql_reset_connection 来重新初始化连接资源。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态。

#### 1.2.3、查询缓存（8.0后没有此功能）

命中缓存则查询缓存，但是大多数情况下我会建议你不要使用查询缓存，为什么呢？**因为查询缓存往往弊大于利**



#### 1.2.4、分析器



##### 1.2.4.1、词法分析

如果没有命中缓存则开始真正的执行语句，分析器先会做“词法分析”。你输入的是由多个字符串和空格组成的一条 SQL 语句，MySQL 需要识别出里面的字符串分别是什么，代表什么。MySQL 从你输入的"select"这个关键字识别出来，这是一个查询语句。它也要把字符串“T”识别成“表名 T”，把字符串“ID”识别成“列 ID”。

##### 1.2.4.2、语法分析

根据词法分析的结果，语法分析器会根据语法规则，判断你输入的这个 SQL 语句是否满足 MySQL 语法。

例如：

```sql
elect * from where id =1;

ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'elect * from where id =1' at line 1
```



如果你的语句不对，就会收到“You have an error in your SQL syntax”的错误提醒（为**语法分析**问题），比如下面这个语句 select 少打了开头的字母“s”。位置提示是在：一般语法错误会提示第一个出现错误的位置，所以你要关注的是紧接“use near”的内容。

#### 1.2.5、优化器

优化器是在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联（join）的时候，决定各个表的连接顺序。

#### 1.2.6、执行器（权限校验）

开始执行的时候，要先判断一下你对这个表 T 有没有执行查询的权限，如果没有，就会返回没有权限的错误，如下所示 (在工程实现上，如果命中查询缓存，会在查询缓存返回结果的时候，做权限验证。查询也会在优化器之前调用 precheck 验证权限)。



比如我们这个例子中的表 T 中，ID 字段没有索引，那么执行器的执行流程是这样的：

1. 调用 InnoDB 引擎接口取这个表的第一行，判断 ID 值是不是 10，如果不是则跳过，如果是则将这行存在结果集中；
2. 调用引擎接口取“下一行”，重复相同的判断逻辑，直到取到这个表的最后一行。
3. 执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端。

至此，这个语句就执行完成了。

对于有索引的表，执行的逻辑也差不多。第一次调用的是“取满足条件的第一行”这个接口，之后循环取“满足条件的下一行”这个接口，这些接口都是引擎中已经定义好的。你会在数据库的慢查询日志中看到一个 **rows_examined** 的字段，表示这个语句执行过程中扫描了多少行。这个值就是在执行器每次调用引擎获取数据行的时候累加的。

在有些场景下，执行器调用一次，在引擎内部则扫描了多行，因此**引擎扫描行数跟 rows_examined 并不是完全相同的**。



## 2、日志系统：

### 2.1、问题：

1. redo log的概念是什么? 为什么会存在.
2. 什么是WAL(write-ahead log)机制, 好处是什么.
3. redo log 为什么可以保证crash safe机制.
4. binlog的概念是什么, 起到什么作用, 可以做crash safe吗? 
5. binlog和redolog的不同点有哪些? 
6. 物理一致性和逻辑一直性各应该怎么理解? 
7. 执行器和innoDB在执行update语句时候的流程是什么样的?
8. 如果数据库误操作, 如何执行数据恢复?
9. 什么是两阶段提交, 为什么需要两阶段提交, 两阶段提交怎么保证数据库中两份日志间的逻辑一致性(什么叫逻辑一致性)?
10. 如果不是两阶段提交, 先写redo log和先写bin log两种情况各会遇到什么问题?



### 2.2、内容：

#### 2.2.1、redo log（重做日志， redo log 是 InnoDB 引擎特有的日志）

最根本的思路是：**MySQL 里经常说到的 WAL 技术，WAL 的全称是 Write-Ahead Logging，它的关键点就是先写日志，再写磁盘**

具体来说，当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 redo log（粉板）里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做，**InnoDB 的 redo log 是固定大小的**。

##### 2.2.1.1、crash-safe

**主要作用：**InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失

![img](https://static001.geekbang.org/resource/image/16/a7/16a7950217b3f0f4ed02db5db59562a7.png)

write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。

#### 2.2.2、binlog（归档日志，Server 层）

因为最开始 MySQL 里并没有 InnoDB 引擎。MySQL 自带的引擎是 MyISAM，但是 MyISAM 没有 crash-safe 的能力，**binlog 日志只能用于归档**



#### 2.2.3、两种日志的不同之处

1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。



#### 2.2.4、执行update语句的内部流程

##### **2.2.4.1、流程图：**

图中浅色框表示是在 InnoDB 内部执行的，深色框表示是在执行器中执行的。

![img](https://static001.geekbang.org/resource/image/2e/be/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)

**具体流程**：

1. 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
2. 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。



##### 2.2.4.2、两阶段提交

**redo log 和 binlog 都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致。**

### 2.3、总结

mysql的重要的两个日志，物理日志 redo log 和逻辑日志 binlog。

redo log 用于保证 crash-safe 能力。innodb_flush_log_at_trx_commit 这个参数设置成 1 的时候，表示每次事务的 redo log 都直接持久化到磁盘。这个参数我建议你设置成 1，这样可以保证 MySQL 异常重启之后数据不丢失。

**innodb_flush_log_at_trx_commit** **（默认值为1）**

- 要完全符合ACID，必须使用默认设置1。日志在每次事务提交时写入并刷新到磁盘。
- 设置为0时，每秒写入一次日志并将其刷新到磁盘。尚未刷新日志的事务可能会在崩溃中丢失。
- 设置为2时，在每次事务提交后写入日志，并每秒刷新一次到磁盘。尚未刷新日志的事务可能会在崩溃中丢失。

**日志的刷新间隔时间**：**innodb_flush_log_at_timeout（默认值为1）**

日志刷新频率由来控制 [`innodb_flush_log_at_timeout`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_flush_log_at_timeout)，可让您将日志刷新频率设置为 N秒（其中 *`N`*为`1 ... 2700`，默认值为1）。但是，任何 [**mysqld**](https://dev.mysql.com/doc/refman/8.0/en/mysqld.html)进程崩溃都可能擦除多达 N几秒钟的事务。

每秒钟写入并刷新日志*`N`* 。 [`innodb_flush_log_at_timeout`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_flush_log_at_timeout) 允许增加两次刷新之间的超时时间，以减少刷新并避免影响二进制日志组提交的性能。默认设置为 [`innodb_flush_log_at_timeout`](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_flush_log_at_timeout) 每秒一次。

```sql
--查询Redo Log
show variables like 'innodb_flush_log_at_trx_commit';

--修改 innodb_flush_log_at_trx_commit为0、 1、 2
set global innodb_flush_log_at_trx_commit=2;
--修改innodb_flush_log_at_timeout同上

```



**sync_binlog （默认值为1）**

这个参数设置成 1 的时候，表示每次事务的 binlog 都持久化到磁盘。这个参数我也建议你设置成 1，这样可以保证 MySQL 异常重启之后 binlog 不丢失。

控制MySQL服务器将二进制日志同步到磁盘的频率。

- [`sync_binlog=0`](https://dev.mysql.com/doc/mysql-replication-excerpt/8.0/en/replication-options-binary-log.html#sysvar_sync_binlog)：禁用MySQL服务器将二进制日志同步到磁盘的功能。取而代之的是，MySQL服务器依靠操作系统不时地将二进制日志刷新到磁盘上，就像处理其他任何文件一样。此设置提供最佳性能，但是在电源故障或操作系统崩溃的情况下，服务器可能提交了尚未同步到二进制日志的事务。
- [`sync_binlog=1`](https://dev.mysql.com/doc/mysql-replication-excerpt/8.0/en/replication-options-binary-log.html#sysvar_sync_binlog)：（**默认的值**）启用在提交事务之前将二进制日志同步到磁盘。这是最安全的设置，但由于磁盘写入次数增加，可能会对性能产生负面影响。如果出现电源故障或操作系统崩溃，二进制日志中缺少的事务将仅处于准备状态。这允许自动恢复例程回滚事务，从而保证二进制日志中不会丢失任何事务。
- [`sync_binlog=*`N`*`](https://dev.mysql.com/doc/mysql-replication-excerpt/8.0/en/replication-options-binary-log.html#sysvar_sync_binlog)，（最大值为4294967295）其中*`N`*的值不是0或1：是`N`二进制日志提交组已收集之后，二进制日志将同步到磁盘 。在电源故障或操作系统崩溃的情况下，服务器可能提交了尚未刷新到二进制日志的事务。由于磁盘写入次数的增加，此设置可能会对性能产生负面影响。较高的值可以提高性能，但会增加数据丢失的风险。

```sql
--查询sync_binlog状态
show variables like '%sync_binlog%';

--设置sync_binlog状态
set global sync_binlog=0;


```































