---
title: Innodb 锁和事务模型
date: 2021-10-21 21:18:17
tags:
---

## 锁类型
#### 共享和排他锁（Shared and Exclusive Locks）
InnoDB 实现了两种标准的行级锁，即 共享锁(S锁)和排他锁(X锁)

- 共享锁允许持有锁的事务读取一行
- 排他锁允许持有锁的事务更新或删除一行
<!--more-->
如果一个事务 T1 持有行 r 上的共享锁，另一个事务 T2 来请求设置行 r 的锁时:

- T2 的共享锁请求会被立即批准。此时，T1 和 T2 事务都持有 r 上的共享锁。
- T2 的排他锁请求不会被立即批准，要等待 T1 事务释放 r 上的共享锁。

如果事务 T1 持有行 r 上的排他锁，则另一个事务T2对行 r 上任何类型的锁请求都不会被立即批准，必须等待事务 T1 释放行 r 上的排他锁。
#### 意向锁（Intention Locks）
InnoDB 支持多粒度锁，允许表锁和行锁共存。例如：**LOCK TABLES…WRITE **可以给指定表加上排他锁(X锁)。InnoDB 使用意向锁来实现多粒度的锁。意向锁是 **表级别** 的锁，表示一个事务操作表时需要哪种类型的锁（共享或者排他）。意向锁有两种：

- 意向共享锁（IS锁），指一个事务打算在表中的某些行上设置共享锁。
- 意向排他锁（IX锁），指一个事务打算在表中的某些行上设置排他锁。

**SELECT…LOCK IN SHARE MODE **设置行的 IS 锁，**SELECT…FOR UPDATE **设置行的 IX锁。
意向锁的规定如下：

- 事务在获取表中某些行的 S 锁之前，必须先获取表的 IS 锁或者更高级别的锁。
- 事务在获取表中某些行的 X 锁之前，必须先获取表的 IX 锁。

表级锁类型兼容性如下：

| **-** | **X** | **IX** | **S** | **IS** |
| --- | --- | --- | --- | --- |
| X | 冲突 | 冲突 | 冲突 | 冲突 |
| IX | 冲突 | 兼容 | 冲突 | 兼容 |
| S | 冲突 | 冲突 | 兼容 | 兼容 |
| IS | 冲突 | 兼容 | 兼容 | 兼容 |

事务在获取锁时，如果和其他事务已持有的锁能够兼容，可以成功获取，否则需要等待已持有锁释放后才能成功获取。如果锁请求与已持有锁发生冲突，并且可能导致死锁时，则会抛出一个错误。
意图锁不会阻塞任何东西，除了全表类型的请求（例如：LOCK TABLES ... WRITE）。意图锁的主要作用是表明某个事务正在锁定一行或者将要锁定一行
在 **SHOW ENGINE INNODB STATUS **和 INNODB monitor 输出中，意图锁的事务数据类似如下:
> TABLE LOCK table `test`.`t` trx id 10080 lock mode IX

#### 行锁（Record Locks）
行锁是作用在索引记录上的锁。例如 **SELECT c1 FROM t WHERE c1 = 10 For UPDATE; **能防止其他事物插入、更新或删除 t 表 c1=10 的这行记录。
行锁只锁定索引记录，即使这个表没有定义索引。在这种情况下，InnoDB 会创建一个隐藏的**聚簇索引，**并用这个索引来执行行锁。
在 **SHOW ENGINE INNODB STATUS **和INNODB monitor输出中，行锁的事务数据类似如下:
> RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
> trx id 10078 lock_mode X locks rec but not gap
> Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
>  0: len 4; hex 8000000a; asc     ;;
>  1: len 6; hex 00000000274f; asc     'O;;
>  2: len 7; hex b60000019d0110; asc        ;;

#### 间隙锁（Gap Locks）
间隙锁是指作用于索引记录之间的锁，也可以用作第一条索引记录前或者最后一条索引记录后的间隙。例如：
> SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;

这条语句可以防止其他事务插入或修改c1=15，无论列中是否有15这个值，因为这个范围内的间隙被锁住了。
间隙锁的间隙可以跨越单个或多个索引，甚至没有索引。
间隙锁是在性能和并发之间权衡的一种折衷做法，只能在部分事务（可重复读、可串行化）隔离级别下使用。
在使用唯一索引来查找记录行时，不需要间隙锁来锁定行（不包括搜索条件里有多列唯一索引的情况）。例如下面的语句，如果 id 列有唯一索引，那么下面的语句只对 id = 100 的行使用索引记录锁，不用管该记录之前的间隙
> SELECT * FROM child WHERE id = 100;

如果 id 列没有索引或者不是唯一索引，则该语句会锁定 id=100 之前的间隙（**经测试发现也会锁住记录行之后的间隙**）。
还有一点需要注意，不同事务可以持有作用于同个间隙上的冲突的锁。例如，事务A持有持有一个间隙上的共享间隙锁，同时事务B可以持有相同间隙上的排他间隙锁。锁冲突仍可以共存的原因是：如果某个记录从索引中删除时，这条记录上的间隙锁（多个事务持有的）会被合并。
#### 临键锁（Next-Key Locks）
临键锁是行锁和间隙（GAP）锁的结合。InnoDB 行级锁执行流程是这样的：当语句搜索或者扫描表索引时，它会在遇到的索引记录上设置共享锁或者排他锁，所以，行级锁实际上时索引记录锁。索引记录上的临键锁也会影响该索引记录之前的间隙。也就是说，临键锁是索引记录锁加上间隙锁。如果一个会话在索引的记录R上有一个共享锁或者排他锁，那么另一个会话就不能按照索引顺序在R之前的间隙中插入新的记录。
假设一个索引包含值10、11、13和20，该索引可能有以下区间的临键锁。（圆括号表示排除端点值，方括号表示包含端点值）：
> (negative infinity, 10]
> (10, 11]
> (11, 13]
> (13, 20]
> (20, positive infinity)

默认情况下，InnoDB运行在可重复读的事务隔离级别，此时，InnoDB使用临键锁来搜索和扫描索引，可以防止出现幻读（一个事务内多次读取，数据总量不一致）。
在SHOW ENGINE INNODB STATUS和INNODB monitor输出中，临键锁的事务数据类似如下:
> RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
> trx id 10080 lock_mode X
> Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
>  0: len 8; hex 73757072656d756d; asc supremum;;
> 
> Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
>  0: len 4; hex 8000000a; asc     ;;
>  1: len 6; hex 00000000274f; asc     'O;;
>  2: len 7; hex b60000019d0110; asc        ;;


## 事务隔离级别
事务隔离是数据库处理数据的基础。隔离是数据库ACID（原子性、一致性、隔离性、持久性）中的I。隔离级别是在多个事务同时对数据进行更新和查询时，对性能和结果一致性之间的平衡进行调整的设置。
InnoDB提供了 SQL:1992 标准中描述的四个事务隔离级别：**READ UNCOMMITTED**、**READ COMMITTED**、 **REPEATABLE READ** 和 **SERIALIZABLE**。
#### 可重复读（REPEATABLE READ）
InnoDB的默认隔离级别。同一个事务中的查询会读取第一次查询建立的快照。意思是，如果在同一个事务中执行几次普通（不加锁）SELECT语句，这些语句读取到的结果是一致的。
对于加锁查询（**SELECT with FOR UPDATE or LOCK IN SHARE MODE）、UPDATE**和**DELETE语句，**锁定范围取决于语句的查询条件是使用唯一索引查找特定记录还是范围搜索。使用唯一索引进行搜索的语句，InnoDB只锁定查找到的索引记录，对于其他搜索条件，InnoDB 使用间隙锁或者next-key锁来阻塞其他事务插入记录到搜索范围覆盖的间隙
#### 读已提交（READ COMMITTED）
在同一个事务中，每次一致读都会设置并读取新快照。
对于锁定读取（SELECT with FOR UPDATE or LOCK IN SHARE MODE）、UPDATE语句和DELETE语句，InnoDB只锁定索引记录，不会锁定他们之前的间隙，所以允许在锁定的记录旁边插入新记录。间隙锁只用于外键约束检查和重复键检查。
禁用了间隙锁，导致可能会出现幻读，因为其他会话可以将新行插入间隙中。
对于UPDATE和DELETE语句，InnoDB只对他更新或者删除的行持有锁，不匹配的行记录锁在计算WHRER条件之后被释放，这大大降低了发生死锁的可能性。
#### 读未提交（READ UNCOMMITTED）
SELECT语句以不加锁的方式执行，但是较早版本中可能不同。使用此隔离级别时，读取可能是不一致的，也称为脏读。另外，此隔离级别的工作方式类似于读已提交。
#### 串行读（SERIALIZABLE）
这个级别类似于可重复读，如果禁用了自动提交，InnoDB隐式地将所有普通SELECT语句转换为SELECT ... LOCK IN SHARE MODE ，如果启用了自动提交，则 SELECT 自己作为一个事务。这种隔离级别数据更加安全，但是并发能力较差。

> **SELECT** @@transaction_isolation;    -- 查询隔离级别
> **SET SESSION TRANSACTION ISOLATION LEVEL **_level_;   -- 设置会话隔离级别

## 死锁
死锁是指不同事务无法继续执行的情况。每个事务都持有另一个事务需要的锁，都在等待资源可用，都不会释放自己所持有的锁。
当事务锁住多个表中的行时（通过UPDATE 或者 SELECT ... FOR UPDATE），但是顺序相反，也可能发生死锁。因为时间顺序不同，每个事务都锁定某些索引记录或者间隙，但是没有获得其他锁。
减少发生死锁的可能性，可以采取以下优化：

- 使用事务替代LOCK TABLES语句
- 保持事务小和执行时间短
- 当不同事务更新多个表或者大范围的行时，保持相同的操作顺序
- SELECT ... FOR UPDATE 和 UPDATE ... WHERE 使用的列上创建索引
- 加锁语句放事务最后
- 


当死锁检测（innodb_deadlock_detect）启用（默认）时发生死锁，InnoDB会回滚其中一个事务。如果禁用了死锁检测，则事务回滚依赖于innodb_lock_wait_timeout设置。
使用**SHOW ENGINE InnoDB STATUS**可以查看用户事务最后一次死锁信息。启用**innodb_print_all_deadlocks**会将所有死锁信息打印到**mysqld**错误日志中。
#### 示例
> -- clicnt A
> mysql> CREATE TABLE t (i INT) ENGINE = InnoDB;
> Query OK, 0 rows affected (1.07 sec)
> 
> mysql> INSERT INTO t (i) VALUES(1);
> Query OK, 1 row affected (0.09 sec)
> 
> mysql> START TRANSACTION;
> Query OK, 0 rows affected (0.00 sec)
> 
> mysql> SELECT * FROM t WHERE i = 1 LOCK IN SHARE MODE;
> +------+
> | i           |
> +------+
> |          1  |
> +------+

会话A，获取 i=1 的共享锁

> -- client B
> mysql> START TRANSACTION;
> Query OK, 0 rows affected (0.00 sec)
> 
> mysql> DELETE FROM t WHERE i = 1;

会话B执行删除需要排他锁，由于跟会话A持有的共享锁不兼容，锁请求进入锁队列等待同时会话B阻塞。

> -- client A
> mysql> DELETE FROM t WHERE i = 1;
> ERROR 1213 (40001): Deadlock found when trying to get lock;
> try restarting transaction

此时发生死锁。会话A需要排他锁来删除行，但是，由于会话B已经有一个排他锁请求在队列并且等待会话A释放共享锁，所以会话A的锁请求不能被允许。而且因为会话B先请求排他锁，所以会话A的共享锁也不能升级成排他锁。最终，InnoDB选择某个会话生成死锁错误并释放持有的锁。


