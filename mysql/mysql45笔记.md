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



## 3、事务隔离

### 3.1、问题：

1. 事务的概念是什么?
2. mysql的事务隔离级别读未提交, 读已提交, 可重复读, 串行各是什么意思?
3. 
  读已提交, 可重复读是怎么通过视图构建实现的?
4. 
  可重复读的使用场景举例? 对账的时候应该很有用?
5. 
  事务隔离是怎么通过read-view(读视图)实现的?
6. 
  并发版本控制(MCVV)的概念是什么, 是怎么实现的?
7. 
  使用长事务的弊病? 为什么使用常事务可能拖垮整个库?
8. 
  事务的启动方式有哪几种? 
9. 
  commit work and chain的语法是做什么用的? 
10. 
  怎么查询各个表中的长事务?
11. 
   如何避免长事务的出现?



### 3.2、内容：

#### 3.2.1、隔离级别

在谈隔离级别之前，你首先要知道，你隔离得越严实，效率就会越低。因此很多时候，我们都要在二者之间寻找一个平衡点。SQL 标准的事务隔离级别包括：读未提交（read uncommitted）、读提交（read committed）、可重复读（repeatable read）和串行化（serializable ）。下面我逐一为你解释：

- 读未提交是指，一个事务还没提交时，它做的变更就能被别的事务看到。
- 读提交是指，一个事务提交之后，它做的变更才会被其他事务看到。
- 可重复读是指，一个事务执行过程中看到的数据，总是跟这个事务在启动时看到的数据是一致的。当然在可重复读隔离级别下，未提交变更对其他事务也是不可见的。
- 串行化，顾名思义是对于同一行记录，“写”会加“写锁”，“读”会加“读锁”。当出现读写锁冲突的时候，后访问的事务必须等前一个事务执行完成，才能继续执行。

**总结：**

- 读未提交：别人改数据的事务尚未提交，我在我的事务中也能读到。
- 读已提交：别人改数据的事务已经提交，我在我的事务中才能读到。
- 可重复读：别人改数据的事务已经提交，我在我的事务中也不去读。
- 串行：我的事务尚未提交，别人就别想改数据。

这4种隔离级别，并行性能依次降低，安全性依次提高。

```sql
事务隔离级别						脏读			不可重复读		 幻读

读未提交（READ UNCOMMITTED）		 是				是			  是
读已提交（READ COMMITTED）					     是			   是
可重复读（REPEATABLE READ）									    是
串行化
--查询当前会话的事物隔离级别
SELECT @@transaction_isolation;
--查询系统的当前会话事物隔离级别
select @@global.transaction_isolation;

--读未提交事务（修改系统的把session 改为global）
set session transaction isolation level read uncommitted;

--读已提交
set session transaction isolation level read committed;
--可重复读
set session transaction isolation level repeatable read;
```



**注意：**

Oracle 数据库的默认隔离级别其实就是“读提交”，因此对于一些从 Oracle 迁移到 MySQL 的应用，为保证数据库隔离级别的一致，你一定要记得将 MySQL 的隔离级别设置为“**读提交**”。

#### 3.2.2、事务隔离级别的实现

InnoDB的MVCC，是通过每行记录后面的保存的两个隐藏的列来实现的。一个是保存了行的创建时间，一个是保存行的过期时间（或删除时间）。当然存储的并不是实际的时间值而是系统的版本号。每开始一个新的事务，系统版本都会自动递增。事务开始的时刻的系统版本号会作为事务的版本号，用来和查询到的每行记录的版本号进行比较。



#### 3.2.3、事务的启动方式

MySQL 的事务启动方式有以下几种：

1. 显式启动事务语句， begin 或 start transaction。配套的提交语句是 commit，回滚语句是 rollback。
2. set autocommit=0，这个命令会将这个线程的自动提交关掉。意味着如果你只执行一个 select 语句，这个事务就启动了，而且并不会自动提交。这个事务持续存在直到你主动执行 commit 或 rollback 语句，或者断开连接。

**导致长事务的原因：**

有些客户端连接框架会默认连接成功后先执行一个 set autocommit=0 的命令。这就导致接下来的查询都在事务中，如果是长连接，就导致了意外的长事务。



**默认情况下，客户端连接以[`autocommit`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_autocommit)设置为1 开始**，如果设置为0，则必须使用 [`COMMIT`](https://dev.mysql.com/doc/refman/8.0/en/commit.html)接受交易或[`ROLLBACK`](https://dev.mysql.com/doc/refman/8.0/en/commit.html) 取消交易。如果[`autocommit`](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_autocommit) 为0，并将其更改为1，则MySQL会自动执行 [`COMMIT`](https://dev.mysql.com/doc/refman/8.0/en/commit.html)所有未清事务

```sql
--设置提交事务方式
--全局的设置
[mysqld]
autocommit=0

--会话的设置
SET autocommit=0;
```



**解决方法：**

使用 set autocommit=1 的情况下，用 begin 显式启动的事务，如果执行 commit 则提交事务。如果执行 commit work and chain，则是**提交事务并自动启动下一个事务**，这样也省去了再次执行 begin 语句的开销。同时带来的好处是从程序开发的角度明确地知道每个语句是否处于事务中。（如果执行commit work and chain会在每个事务开始时少一次begin）

```sql
--启动显示事务的命令
BEGIN [WORK]
COMMIT [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
ROLLBACK [WORK] [AND [NO] CHAIN] [[NO] RELEASE]
SET autocommit = {0 | 1}

--你可以在 information_schema 库的 innodb_trx 这个表中查询长事务，比如下面这个语句，用于查找持续时间超过 60s 的事务。

select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60

```



## 4、索引

### 4.1、问题

1. innodb B+树主键索引的叶子节点存的是什么？
   B+树的叶子节点是page （页），一个页里面可以存多个行

   

2. **“N叉树”的N值在MySQL中是可以被人工调整的么？**
   N 叉”树中的“N”取决于数据块的大小。可以按照调整key的大小的思路来说；
   如果你能指出来5.6以后可以通过page大小来间接控制应该能加分吧，

   **总体来说：N是由页大小和索引大小决定的**

   

3. 请问没有主键的表，有一个普通索引。怎么回表？
   没有主键的表，innodb会给默认创建一个Rowid做主键

   

4. 你可以想象一下一棵 100 万节点的平衡二叉树，树高 20。一次查询可能需要访问 20 个数据块。为什么树高20就是20个数据块？

   每个叶子结点就是一个块，每个块包含两个数据，块之间通过链式方式链接。树高20的话，就要遍历20个块。

   

5. 如果插入的数据是在主键树叶子结点的中间，后面的所有页如果都是满的状态，是不是会造成后面的每一页都会去进行页分裂操作，直到最后一个页申请新页移过去最后一个值？

   不会不会，只会分裂它要写入的那个页面。每个页面之间是用指针串的，改指针就好了，不需要“后面的全部挪动

   

6. 还有之前看到过说是插入数据如果是在某个数据满了页的首尾，为了减少数据移动和页分裂，会先去前后两个页看看是否满了，如果没满会先将数据放到前后两个页上，不知道是不是有这种情况？

   对，减为了增加空间利用率

   

7. 每一个表是好几棵B+树，树结点的key值就是某一行的主键，value是该行的其他数据。新建索引就是新增一个B+树，查询不走索引就是遍历主B+树。

   

8. 数据块和数据页是同一个东西吗？16k的这个数据页和数据块有什么区别？
   **如果前缀是InnoDB， 是指同一个东西**

   

9. 如果是组合主键，底层机制和原理 和 普通索引一样吗？（包括组合主键和组合索引）

   

10. 主键为4个字节 int 类型的情况下, 非叶子节点可以存储 1204 个键, 但我仍然无法得知 这个 1204 是怎么的出来的？
    整型4个字节，加上辅助数据差不多每个key占13字节，16k/13





1. **今天这个 alter table T engine=InnoDB , InnoDB 这种引擎，虽然删除了表的部分记录,但是它的索引还在, 并未释放.只能是重新建表才能重建索引.**





### 4.2、内容：

**意义：索引的出现其实就是为了提高数据查询的效率，就像书的目录一样。**



#### 4.2.1、索引的常见模型

- **哈希表：**

  哈希表这种结构适用于只有等值查询的场景

  哈希冲突的处理方法：链表

  

- **有序数组：**

  有序数组索引只适用于静态存储引擎

  查询快，更新慢

  

- **二叉搜索树：**

  每个节点的左儿子小于父节点，父节点又小于右儿子，

  查询时间复杂度O(log(N))，更新时间复杂度O(log(N))

#### 4.2.3、索引模型

**InnoDB 使用了 B+ 树索引模型，**所以数据都是存储在 B+ 树中的。每一个索引在 InnoDB 里面对应一棵 B+ 树。

**索引类型：**

分为主键索引和非主键索引。

- 主键索引的叶子节点存的是整行数据。在 InnoDB 里，主键索引也被称为**聚簇索引**（clustered index）。**整张表的数据其实就是存在主键索引中的**
- 非主键索引的叶子节点内容是主键的值。在 InnoDB 里，非主键索引也被称为**二级索引**（secondary index）。

**索引的区别：**

（1）如果语句为select * from T where ID=500, 主键索引，只需要搜索ID这个B+树
（2）如果语句为select * from T where k = 5 , 普通索引，先查询k这个B+树，然后得到id的值，再搜索ID这个B+树，这个过程叫做回表
**非主键索引需要多扫描一棵索引树，所以尽量用主键索引**

主键索引和普通索引的区别：主键索引只要搜索ID这个B+Tree即可拿到数据。普通索引先搜索索引拿到主键值，再到主键索引树搜索一次(回表)

#### 4.2.4、索引的维护

由于每个非主键索引的叶子节点上都是主键的值。如果用身份证号做主键，那么每个二级索引的叶子节点占用约 20 个字节，而如果用整型做主键，则只要 4 个字节，如果是长整型（bigint）则是 8 个字节。

**显然，主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。**

**B+ 树为了维护索引有序性的内部变化：**

一个数据页满了，按照B+Tree算法，新增加一个数据页，叫做页分裂，会导致性能下降。空间利用率降低大概50%。当相邻的两个数据页利用率很低的时候会做数据页合并，合并的过程是分裂过程的逆过程。

主键索引的应用场景满足的条件：

1. **只有一个索引；**
2. **该索引必须是唯一索引。**























