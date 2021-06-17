## 锁概述

当事务更新表中的一行时，或用`SELECT FOR UPDATE`锁定它时，InnoDB 会在该行上建立一个锁列表或队列。类似地，InnoDB 在表上维护表级锁的锁列表。如果第二个事务想更新一行，或与前一个事务锁定同一个的表，但锁定方式互斥，InnoDB 会将该行的锁请求添加到相应的队列中。事务要想获取到锁，必须等先前进入该行或表的锁队列中的所有不兼容的锁请求都释放后（持有或请求锁的事务提交或回滚）。

### 行级锁定

MySQL 对 InnoDB 表使用行级锁定，以支持多个会话同时进行写访问。

为避免在单个InnoDB表上执行多个写操作时出现死锁，即使数据变更语句出现在事务后期，也最好在事务开始时通过SELECT ... FOR UPDATE 对要修改的每组的数据行进行锁定。如果事务修改或锁定多个表，则需要在每个事务中以相同的顺序执行语句。死锁会影响性能，但不是会造成严重错误，因为 InnoDB 会自动检测死锁，并回滚一个受影响的事务。

在高并发系统，多个线程竞争同一个锁时，死锁检测可能会导致性能降低。有时，禁用死锁检测并在死锁发生时，依赖innodb_lock_wait_timeout 配置事务锁等待超时可能更有效。可以使用 innodb_deadlock_detect 配置禁用死锁检测。

行级锁定的优点：

- 当不同会话存取不同行时，锁竞争概率小和冲突更少；
- 回滚改动更少；
- 可以长时间锁定一行；

行级锁定的缺点：

- 比页级或表级锁定占用更多的内存。
- 当在表的大部分中使用时，比页级或表级锁定速度慢，因为你必须获取更多的锁。
- 如果你在大部分数据上经常进行GROUP BY操作或者必须经常扫描整个表，比其它锁定明显慢很多。

#### 共享锁和独占锁（Shared and Exclusive Locks）

InnoDB支持行锁，有两种类型：

- 共享锁（shared locks，简称 S）：允许持有锁的事务读取一行数据；
- 独占锁（exclusive locks，简称 X）：允许持有锁的事务更新或删除一行数据；

如果事务 T1 持有行 r 的共享锁，然后来了另一个事务 T2，也想获取行 r 的锁，处理方式如下：

- 如果事务 T2 获取的是共享锁，则直接授予，结果就是T1 T2 都持有行 r 的共享锁；
- 如果事务 T2 获取的是独占锁，则必须等待，直到可以独占行 r 时；

多个事务间，对一行数据，都持有共享锁时可并行操作，否则必须串型操作。共享锁可同时作用于一行，共享锁和独占锁、独占锁与独占锁不能同时作用于一行，必须依次执行。

#### 记录锁（Record Locks）

记录锁是建立在索引记录上的锁。例如：`SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;`防止其他事务更新 c1 为 10 的所有行。

记录锁建立在索引上。即使表中没有自定义索引，对于这种情况，InnoDB 会创建一个隐藏聚簇索引，用来做记录锁定。

可使用命令`SHOW ENGINE INNODB STATUS`查看 InnoDB  监控信息，其中记录锁的事务数据类似如下示例：

```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

#### 间隙锁（Gap Locks）

间隙锁是对索引记录之间间隙的锁定（不包括记录本身），或者是对第一个索引记录之前，最后一个索引记录之后的间隙的锁定。例如：`SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE;`防止其他事务将值 15 插入到列 c1 中，无论该列中是否已有任何此值，因为该范围内的间距都已锁定。

间隙可以是单个索引值、多个索引值，甚至可以是空的。

间隙锁是在性能和并发性之间进行权衡，仅用于某些事务隔离级别中。

对于使用唯一索引搜索唯一数据行的语句，不需要使用间隙锁（如果是复合唯一索引，但查询条件仅为部分列，则依然使用间隙锁）。例如，如果 id 列具有唯一索引，则下面的语句仅对 id 值为 100 的行使用记录锁，不影响其他事务在前面的间隙插入数据。

```sql
SELECT * FROM child WHERE id = 100;
```

如果 id 列没有索引或索引不唯一，则语句会锁定前面的间隙。

```java
// FIXME 为什么间隙锁锁住前面的间隙？会锁住后面的间隙吗？
```

值得注意的是，不同的事务可以保留冲突的锁在间隙上。例如，事务 A 可以在一个 gap 上持有一个共享的 gap 锁（gap S-lock），而事务 B 在同一个 gap 上持有一个独占的gap锁（gap X-lock）。允许冲突的间隙锁的原因是，如果从索引中清除记录，则必须合并由不同事务保留在记录上的间隙锁。

间隙锁唯一目的是防止其他事务插入到间隙中。多个间隙锁可以共存。一个事务的间隙锁并不阻止另一个事务在同一间隙上获取间隙锁。共享和独占间隙锁之间没有区别。它们彼此不冲突，并且执行相同的功能。

如果将事务隔离级别更改为 READ COMMITTED，则会禁用间隙锁。在这种情况下，对搜索和索引扫描禁用间隙锁定，并且仅用于外键约束检查和重复键检查。

使用 READ-COMMITTED 隔离级别还有其他效果。在 MySQL 评估 WHERE 条件之后，将释放不匹配行的记录锁。对于UPDATE 语句，InnoDB 执行 `semi-consistent` 读取，这样它会将最新提交的版本返回给 MySQL，以便 MySQL 可以确定行是否匹配更新的 WHERE 条件。

#### 临键锁（Next-Key Locks）

临键锁是索引记录上的记录锁和索引记录之前的间隙的间隙锁的组合。

InnoDB执行行级锁定，其方式是在搜索或扫描表索引时，对遇到的索引记录设置共享或独占锁。因此，行级锁实际上是索引记录锁。索引记录上的临键锁也会影响该索引记录之前的间隙。也就是说，临键锁是索引记录锁，加上索引记录前面的间隙锁。如果一个会话在索引中的记录 R 上具有共享或独占锁，则另一个会话不能在索引顺序中 R 之前的间隙中插入新的索引记录。

假设索引包含值10、11、13和20。此索引的临键锁可能包含以下间隔，其中圆括号表示排除间隔端点，方括号表示包含端点：

```
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```

对于最后一个间隔，临键锁锁定索引中最大值前面的间隙和“上限”伪记录，该伪记录的值高于索引中的任何实际值。上限不是真正的索引记录，因此实际上，临键锁只锁定最大索引值后面的间隙。

默认情况下，InnoDB 在 REPEATABLE READ 隔离级别下运行。在这种情况下，InnoDB使用临键锁进行搜索和索引扫描，从而防止幻行。

可使用命令`SHOW ENGINE INNODB STATUS`查看 InnoDB  监控信息，其中临键锁的事务数据类似如下示例：

```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

#### 插入意向锁（Insert Intention Locks）

插入意图锁是在插入数据之前，由插入操作设置的一种间隙锁。此锁表示插入的意图，如果插入到同一索引间隙中的多个事务不在间隙中的同一位置插入，则它们不互相阻碍。假设存在值为4和7的索引记录。分别尝试插入值为5和6的单独事务，在获得插入行的独占锁之前，每个事务都使用插入意图锁锁定4和7之间的间隔，但不会相互阻碍，因为行不冲突。

下面的示例演示了一个事务，它在获取插入记录上的独占锁之前使用插入意图锁。这个例子涉及两个客户端，A和B。

客户端A创建一个包含两个索引记录（90和102）的表，然后启动一个事务，在ID大于100的索引记录上放置一个独占锁。独占锁包括一个在记录102之前的间隙锁：

```sql
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```

客户端B开始一个事务，将一条记录插入到间隙。事务持有插入意图锁，然后等待获取独占锁。

```sql
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```

可使用命令`SHOW ENGINE INNODB STATUS`查看 InnoDB  监控信息，其中插入意图锁的事务数据类似如下示例：

```sql
RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...
```

### 表级锁定

对于MyISAM， MEMORY或者MERGE表，使用的是表锁，这种类型的表使用场景多为read-only, read-mostly或者single-user应用中。在获取表锁的时候，如果多个请求需要获取同一张表的锁，那么通常就需要进行排序，当前请求必须等到前一个请求处理完成后才能获取到表锁进行后续的处理。

表级锁定的优点：

- 所需内存相对较少（行锁每行或每组锁定的行都需要内存）；
- 当处理表中大部分行时效率更高，因为只涉及一个锁；
- 如果经常对大部分数据执行分组操作，或者必须经常全表扫描，则速度很快；

获取写锁：

1. 如果没有锁在表上, 放一个写锁；
2. 否则,放一个锁请求在写锁队列中；

获取读锁：

1. 如果没有写锁在表上,放一个读锁；
2. 否则, 放一个锁请求在读锁队列中；

更新操作优先级高于查询操作。因此，当释放锁时，写锁操作先执行，然后是读锁请求。确保了表有大量的查询时，对表的更新不会造成“饥饿问题”。如果一个表有许多更新，SELECT语句将等待，直到不再更新为止。

可以通过命令 `SHOW STATUS LIKE 'Table%';` 检查 Table_locks_immediate 和 Table_locks_waited 状态变量来分析系统的表锁争用，这两个变量分别表示可以立即授予表锁请求的次数和必须等待的次数。

如果使用`LOCK TABLES`显式获取表锁，则可以请求`READ LOCAL`锁而不是读锁，以便在锁定表时使其他会话能够执行并发插入。

若要在无法并发插入时对表 t1 执行大批量插入和查询操作，可以将数据插入临时表 temp_t1 ，并使用临时表中的数据更新表：

```sql
mysql> LOCK TABLES t1 WRITE, temp_t1 WRITE;
mysql> INSERT INTO t1 SELECT * FROM temp_t1;
mysql> DELETE FROM temp_t1;
mysql> UNLOCK TABLES;
```

#### 意向锁（Intention Locks）

InnoDB支持多粒度锁，允许行锁和表锁共存。例如，语句 `LOCK TABLES ... WRITE` 指定的表上采用独占锁（X锁）。为了实现多粒度级别的锁定，InnoDB使用了意向锁。意向锁是表级锁，指明事务稍后在该表中对某一行使用哪类的锁定（共享还是独占）。

- 意向共享锁（intention shared lock，简称 IS）：表示事务打算对表中的特定的行使用共享锁；
- 意向独占锁（intention exclusive lock，简称 IX）：表示事务打算对表中的特定的行使用独占锁；

示例： `SELECT ... FOR SHARE`  设置的是意向共享锁，`SELECT ... FOR UPDATE` 设置的是意向独占锁。

意向锁协议如下：

- 在事务获取表中某行的共享锁之前，必须先获取表上的意向共享锁或更强的锁。
- 在事务获取表中某行的独占锁之前，必须先获取表上的意向独占锁。

下表总结了表级锁类型兼容性：

|                      | 独占锁（X） | 意向独占锁（IX） | 共享锁（S） | 意向共享锁（IS） |
| :------------------: | :---------: | :--------------: | :---------: | :--------------: |
|   **独占锁（X）**    |             |                  |             |                  |
| **意向独占锁（IX）** |             |       兼容       |             |       兼容       |
|   **共享锁（S）**    |             |                  |    兼容     |       兼容       |
| **意向共享锁（S）**  |             |       兼容       |    兼容     |       兼容       |

意向锁之间都兼容，共享锁之间都兼容，其他都不兼容，会产生冲突。

如果请求事务的锁与已有的锁兼容，则将锁授予该事务，但如果与已有锁冲突，则不授予。事务将等待产生冲突的已有锁被释放。如果锁请求与已有锁冲突，则不能授予，因为它会导致死锁，服务发生错误。

意向锁不会阻塞任何请求，除整表锁定请求（eg： `LOCK TABLES ... WRITE`）以外。意向锁的主要目的是显示有人正在锁定一行，或者将要锁定表中的一行。

可使用命令`SHOW ENGINE INNODB STATUS`查看 InnoDB  监控信息，其中意向锁的事务数据类似如下示例：

```sql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

#### 自增锁（AUTO-INC Locks）

自增锁是一种特殊的表级锁，由插入到具有自动增量列的表中的事务使用。在最简单的情况下，如果一个事务正在向表中插入值，则其他向该表中插入值的事务都必须等待，以便第一个事务插入的行接收连续的主键值。

 `innodb_autoinc_lock_mode` 变量控制用于自动增量锁定的算法。它允许您选择如何在可预测的自动增量值序列和插入操作的最大并发性之间进行权衡。

更多信息，请查看章节 [InnoDB 处理自增](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)。

## 选择锁定类型

表锁优于行锁的场景：

1. 表中大部分数据进行读取；

2. 对严格的关键字进行混合读写，其中写是对一行的更新或删除时；

   ```sql
   UPDATE tbl_name SET column=value WHERE unique_key_col=key_value;
   DELETE FROM tbl_name WHERE unique_key_col=key_value;
   ```

3. 带有多个并发写入的查询语句，也可以有极少数的更新和删除语句。

4. 没有写入操作时，对整个表进行多次扫描或分组操作。

使用更高级别的锁，可以更轻松地调优应用程序，因为锁开销比行级别的锁小：

- 版本控制（Versioning ）：（例如 MySQL中用于并发插入的版本）可以允许同时存在一个写和多个读，这意味这数据库或者表支持不同的数据视图，具体数据视图取决于访问时间点，其他常见的术语有："time travel", "copy on write", "copy on demand"等。
- 按需复制：在大多数情况下是比行锁要好的，但是在极端情况下，要比普通的锁占用更多的内存。
- 也可以使用应用级别的锁，使用 GET_LOCK() and RELEASE_LOCK()。这些锁是建议锁，因此它们只适用于相互协作的应用程序。

## 总结

InnoDB表使用行级锁定，这样多个会话和应用程序可以同时读写同一个表，而不会让彼此等待或产生不一致的结果。对于这个存储引擎，请避免使用LOCK TABLES语句，因为它不提供任何额外的保护，而是降低了并发性。自动行级锁定使这些表适用于包含最重要数据的最繁忙数据库，同时也简化了应用程序逻辑，因为不需要锁定和解锁表。因此，InnoDB存储引擎是MySQL中的默认引擎。

MySQL对除InnoDB之外的所有存储引擎都使用表锁定（而不是页、行或列锁定）。锁定操作本身没有太多开销。但是，由于在任何时候只有一个会话可以写入一个表，因此为了获得这些其他存储引擎的最佳性能，请将它们主要用于经常查询、很少插入或更新的表。

在选择是使用InnoDB还是使用其他存储引擎创建表时，请记住表锁定的以下缺点：

- 表锁定允许多个会话同时从表中读取数据，但如果会话要向表中写入数据，则必须首先获得独占访问权限，这意味着它可能必须等待其他会话先完成表的访问。在更新期间，要访问此特定表的所有其他会话必须等待更新完成。
- 当会话正在等待时，表锁定会导致问题，因为磁盘已满，在会话可以继续之前，需要有可用空间。在这种情况下，所有要访问问题表的会话都将处于等待状态，直到有更多的磁盘空间可用。
- 运行SELECT语句需要很长时间，这会阻止其他会话同时更新表，从而使其他会话看起来很慢或没有响应。当会话等待以独占方式访问表以进行更新时，发出SELECT语句的其他会话将在其后面排队，从而降低了并发性，即使对于只读会话也是如此。

## 参考资料

- 官方文档 [15.7.1 InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- 官方文档 [15.15.2.2 InnoDB Lock and Lock-Wait Information](https://dev.mysql.com/doc/refman/8.0/en/innodb-information-schema-understanding-innodb-locking.html)
- 官方文档 [8.11 Optimizing Locking Operations](https://dev.mysql.com/doc/refman/8.0/en/locking-issues.html)

