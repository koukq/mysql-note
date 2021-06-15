## 锁概述

当事务更新表中的一行时，或用`SELECT FOR UPDATE`锁定它时，InnoDB会在该行上建立一个锁列表或队列。类似地，InnoDB在表上维护表级锁的锁列表。如果第二个事务想更新一行或以不兼容模式锁定一个已被前一个事务锁定的表，InnoDB会将该行的锁请求添加到相应的队列中。对于要由事务获取的锁，必须删除先前输入到该行或表的锁队列中的所有不兼容的锁请求（在持有或请求这些锁的事务提交或回滚时发生）。

### 行锁

#### 共享锁和独占锁（Shared and Exclusive Locks）

InnoDB支持行锁，有两种类型：

- 共享锁（shared locks，简称 S）：允许持有锁的事务读取一行数据；
- 独占锁（exclusive locks，简称 X）：允许持有锁的事务更新或删除一行数据；

如果事务 T1 持有行 r 的共享锁，然后来了另一个事务 T2，也想获取行 r 的锁，处理方式如下：

- 如果事务 T2 获取的是共享锁，则直接授予，结果就是T1 T2 都持有行 r 的共享锁；
- 如果事务 T2 获取的是独占锁，则必须等待，直到可以独占行 r 时；

多个事务间，对一行数据，都持有共享锁时可并行操作，否则必须串型操作。共享锁可同时作用于一行，共享锁和独占锁、独占锁与独占锁不能同时作用于一行，必须依次执行。

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

#### 自增锁（AUTO-INC Locks）

自增锁是一种特殊的表级锁，由插入到具有自动增量列的表中的事务使用。在最简单的情况下，如果一个事务正在向表中插入值，则其他向该表中插入值的事务都必须等待，以便第一个事务插入的行接收连续的主键值。

 `innodb_autoinc_lock_mode` 变量控制用于自动增量锁定的算法。它允许您选择如何在可预测的自动增量值序列和插入操作的最大并发性之间进行权衡。

更多信息，请查看章节 [InnoDB 处理自增](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)。

#### 空间索引的断言锁（Predicate Locks for Spatial Indexes）

InnoDB支持对包含空间数据的列建立空间索引。

要处理涉及空间索引操作的锁定，临键锁无法很好地支持`REPEATABLE READ` 或 `SERIALIZABLE` 事务隔离级别。多维数据中没有绝对的排序概念，因此不清楚哪个是下一个键。

为了支持具有空间索引的表的隔离级别，InnoDB使用断言锁。空间索引包含最小边界矩形（MBR）值，因此InnoDB通过对用于查询的MBR值设置断言锁来强制索引的一致读取。其他事务不能插入或修改与查询条件匹配的行。

## 参考资料

