## 事务概述

事务一般是指要做的或所做的事情。在关系数据库中，一个事务可以是一条SQL语句，一组SQL语句或整个程序，是恢复和并发控制的基本单位。事务应该具有4个属性：原子性、一致性、隔离性、持久性，这四个属性通常称为ACID特性。

- A（atomicity）：原子性。一个事务要么全部提交成功，要么全部失败回滚，不能只执行其中的一部分操作，这就是事务的原子性。
- C（consistency）：一致性。事务的执行不能破坏数据库数据的完整性和一致性，一个事务在执行之前和执行之后，数据库都必须处于一致性状态。
- I（isolation）：隔离性。事务的隔离性是指在并发环境中，并发的事务时相互隔离的，一个事务的执行不能不被其他事务干扰。不同的事务并发操作相同的数据时，每个事务都有各自完成的数据空间，即一个事务内部的操作及使用的数据对其他并发事务时隔离的，并发执行的各个事务之间不能相互干扰。
- D（durability）：持久性。一旦事务提交，那么它对数据库中的对应数据的状态的变更就会永久保存到数据库中。

## Mysql 事务

innodb 所有的用户活动都处于事务的模式。

当连接到会话时，默认启用自动提交（即 autocommit = 1），即执行的每条SQL都算一个事务。

```sql
mysql> SHOW VARIABLES LIKE 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set, 1 warning (0.00 sec)
```

可使用 `BEGIN` 或 `START TRANSACTION` 手动开启一个事务之后，自动提交将保持禁用状态，直到使用 `COMMIT` 或 `ROLLBACK` 结束事务。之后，自动提交模式会恢复到之前的状态。

使用 `SET autocommit = 0;` 关闭自动提交。Mysql 会默认开启一个事务，直到 `COMMIT` 或 `ROLLBACK`  执行后，其他会话才能看到本次修改后的数据。而本回话会再次开启一个事务。



```sql
mysql> SET autocommit = 0;
Query OK, 0 rows affected (0.00 sec)

mysql> SHOW VARIABLES LIKE 'autocommit';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | OFF   |
+---------------+-------+
1 row in set, 1 warning (0.00 sec)
```

如何保障事务满足ACID，具体可参考官方文档 [15.2 InnoDB 和 ACID 模型](https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html)。

## 事务隔离

事务隔离是数据库应用的基础之一。隔离是 ACID 模型中的 I ；隔离级别是一种设置，当多个事务同时进行更改和执行查询时，它可以微调性能与结果的可靠性、一致性和可再现性之间的平衡。

InnoDB 提供 SQL:1992 standard 描述的四个事务隔离级别：

- 读未提交（READ UNCOMMITTED，简称 RU）：事务可以看到别的事务未提交的数据（会导致 脏读、不可重复读、幻读）。
- 读提交（ READ COMMITTED，简称 RC）：事务可以看到别的事务提交的数据（会导致 不可重复读、幻读）。
- 可重复读（REPEATABLE READ，简称 RR）：使用 `MMVC`机制 实现可重复读，使用间隙锁和临键锁解决大部分幻行问题。（会导致 幻读）
- 可序列化（SERIALIZABLE）：所有事务有序执行。

> 1. 脏读：读到了其他事务未提交的数据，这些数据可能处于不一致状态，也可能会回滚，最终可能不会存到数据库；
>
> 2. 不可重复读：即一个事务中多次执行同一个查询，可能结果不一致，被别的事务**修改**了；
> 3. 幻读：幻读只针对幻行，即数据**插入**导致的异常现象。

InnoDB的默认隔离级别是可重复读取的。

### 脏读 示例（读未提交 隔离级别）

事务A正在进行购买，事务B在统计销售情况。

已售商品 10，销售额 50。

| 事务A      | 事务B                             |
| ---------- | --------------------------------- |
| 事务开启   |                                   |
|            | 事务开启                          |
| 已售商品+1 |                                   |
|            | 查询 已售商品（11）、销售额（50） |
| 销售额+5   | 事务提交                          |
| 事务提交   |                                   |

结果：导致统计的商品数和销售额匹配不上。

### 不可重复读示例（读提交 隔离级别）

已售商品 10，销售额 50。

| 事务A      | 事务B                             |
| ---------- | --------------------------------- |
| 事务开启   |                                   |
|            | 事务开启                          |
| 已售商品+1 |                                   |
| 销售额+5   | 查询 已售商品（10）、销售额（50） |
| 事务提交   |                                   |
|            | 查询 已售商品（11）、销售额（55） |
|            | 事务提交                          |

结果：一个事务中，多次查询结果不一致。读提交隔离级别可以解决脏读的问题，但还存在不可重复读问题。

### MVCC 实现重复读，特殊幻行事件（可重复读 隔离级别）

InnoDB 是一个多版本并发控制（MVCC ）的存储引擎。它保留有关已更改行的历史版本的信息，以支持事务性功能，如并发和回滚。它还使用这些信息构建数据行的早期版本，以实现一致的读取。

数据的快照（`MVCC`）应用于事务中的 SELECT 语句，而不一定适用于 DML 语句。如果插入或修改某些行并提交该事务，则从另一个 可重复读 事务发出的 DELETE 或 UPDATE 语句可能会影响那些刚刚提交的行，即使会话无法查询它们。如果事务确实更新或删除由其他事务提交的行，那么这些更改对当前事务可见。例如，可能遇到如下情况：

**删除幻行**

| 事务A                               | 事务B                      |
| ----------------------------------- | -------------------------- |
| 事务开启                            |                            |
|                                     | 事务开启                   |
| 查询名字为 张三 的人，结果为 0 个。 |                            |
|                                     | 插入一条名字为 张三 的人。 |
|                                     | 事务提交                   |
| 删除名字 为 张三 的人。结果为 1个。 |                            |
| 事务提交                            |                            |

**更新幻行**

| 事务A                                  | 事务B                  |
| -------------------------------------- | ---------------------- |
| 事务开启                               |                        |
|                                        | 事务开启               |
| 查询年龄大于 18 的人，结果为 0 个。    |                        |
|                                        | 插入一条年龄为30的人。 |
|                                        | 事务提交               |
| 将年龄大于18的人更新为18。结果为 1个。 |                        |
| 查询年龄等于 18 的人，结果为 1 个。    |                        |
| 事务提交                               |                        |

**插入幻行**

名字为唯一索引。

| 事务A                                        | 事务B                      |
| -------------------------------------------- | -------------------------- |
| 事务开启                                     |                            |
|                                              | 事务开启                   |
| 查询名字为 张三 的人，结果为 0 个。          |                            |
|                                              | 插入一条名字为 张三 的人。 |
|                                              | 事务提交                   |
| 插入一条名字为 张三 的人。报错，唯一键重复。 |                            |
| 查询名字为 张三 的人，结果为 0 个。          |                            |

有一条事务A看不到的数据，阻止它插入对应的数据，但它又看不到。

## 只读事务

InnoDB 可以避免为只读事务设置事务ID（`TRX_ID` 字段）带来的开销。只有可能执行写操作或锁定读取（例如：`SELECT ... FOR UPDATE`）的事务才需要事务ID。消除不必要的事务ID，每次查询或数据更改语句构造读视图时，可以减少查询的内部数据结构的大小。

InnoDB 在以下情况下认为是只读事务：

事务是用`START TRANSACTION READ ONLY` 语句启动的。在这种情况下，尝试更改数据库（对于InnoDB、MyISAM或其他类型的表）会导致错误，事务将以只读状态继续：

> ERROR 1792 (25006): Cannot execute statement in a READ ONLY transaction.

仍然可以在只读事务中更改特定于会话的临时表，或者为它们发出锁定查询，因为这些更改和锁定对任何其他事务都不可见。

启用自动提交设置，从而保证事务是单个语句，构成事务的单个语句是“非锁定”的 SELECT 语句。也就是说，一个不使用`FOR UPDATE`或`LOCK IN SHARED MODE`子句的 SELECT。

事务是在没有`READ ONLY`选项的情况下启动的，但是还没有执行显式锁定行的更新或语句。在需要更新或显式锁之前，事务将保持只读模式。

因此，对于生成报表这样的读密集型操作，您可以通过在`START TRANSACTION READ ONLY`和`COMMIT`中对 InnoDB 查询进行优化，或者在运行`SELECT`语句之前打开 `autocommit` 设置，或者简单地避免查询中夹杂的任何数据更改语句，来优化 InnoDB 查询。

## 当前事务查看

`INFORMATION_SCHEMA`.`INNODB_TRX`表提供当前在引擎执行的每个事务的信息，包括事务是否正在等待锁、事务何时启动以及事务正在执行的SQL语句（如果有的话）。

| 字段                       | 描述                                                         |
| -------------------------- | ------------------------------------------------------------ |
| **TRX_ID**                 | InnoDB 内部的唯一事务ID号。只读和非锁定事务没有。            |
| TRX_WEIGHT                 | 事务的权重，反映（但不一定是确切的计数）被更改的行数和被事务锁定的行数。为了解决死锁，InnoDB选择权重最小的事务作为要回滚的“牺牲品”。已更改非事务表的事务被视为比其他事务重，而不管更改和锁定的行数是多少。 |
| **TRX_STATE**              | 事务执行状态。 `RUNNING`, `LOCK WAIT`, `ROLLING BACK`, 和`COMMITTING`. |
| **TRX_STARTED**            | 事务开始时间。                                               |
| TRX_REQUESTED_LOCK_ID      | 如果`TRX_STATE`为`LOCK WAIT`，则事务当前正在等待的锁的ID；否则为空。要获取有关锁的详细信息，请将此列与`performance_schema`.`data_locks`表的`ENGINE_LOCK_ID`列连接起来。 |
| **TRX_WAIT_STARTED**       | 事务开始等待锁的时间，如果`TRX_STATE`为`LOCK WAIT`；否则为空。 |
| TRX_MYSQL_THREAD_ID        | MySQL线程ID。若要获取有关线程的详细信息，请将此列与`PROCESSLIST`表的ID列连接起来。 |
| **TRX_QUERY**              | 事务正在执行的SQL语句。                                      |
| TRX_OPERATION_STATE        | 事务的当前操作（如有）；否则为空                             |
| TRX_TABLES_IN_USE          | 处理此事务的当前 SQL 语句时使用的 InnoDB 表数。              |
| TRX_TABLES_LOCKED          | 当前SQL语句具有行锁定的 InnoDB 表数(因为这些是行锁，而不是表锁，所以尽管某些行被锁定，但通常仍可以由多个事务读取和写入表。） |
| TRX_LOCK_STRUCTS           | 事务保留的锁数。                                             |
| TRX_LOCK_MEMORY_BYTES      | 内存中此事务的锁结构占用的总大小。                           |
| TRX_ROWS_LOCKED            | 此事务锁定的大约行数。该值可能包括物理上存在但事务不可见的删除标记行。 |
| TRX_ROWS_MODIFIED          | 此事务中已修改和插入的行数。                                 |
| TRX_CONCURRENCY_TICKETS    | 一个值，指示当前事务在被调出之前可以做多长时间的工作，由 `innodb_concurrency_tickets` 系统变量指定。 |
| TRX_ISOLATION_LEVEL        | 当前事务的隔离级别。                                         |
| TRX_UNIQUE_CHECKS          | 当前事务唯一性检查启用还是禁用。当批量数据导入时，这个参数是关闭的。 |
| TRX_FOREIGN_KEY_CHECKS     | 当前事务的外键坚持是启用还是禁用。当批量数据导入时，这个参数是关闭的。 |
| TRX_LAST_FOREIGN_KEY_ERROR | 最新一个外键错误信息，没有则为空。                           |
| TRX_ADAPTIVE_HASH_LATCHED  | 自适应哈希索引是否被当前事务阻塞。当自适应哈希索引查找系统分区，一个单独的事务不会阻塞全部的自适应hash索引。自适应hash索引分区通过 `innodb_adaptive_hash_index_parts`参数控制，默认值为8。 |
| TRX_ADAPTIVE_HASH_TIMEOUT  | 是否为了自适应hash索引立即放弃查询锁，或者通过调用mysql函数保留它。当没有自适应hash索引冲突，该值为0并且语句保持锁直到结束。在冲突过程中，该值被计数为0，每句查询完之后立即释放门闩。当自适应hash索引查询系统被分区（由 `innodb_adaptive_hash_index_parts`参数控制），值保持为0。 |
| TRX_IS_READ_ONLY           | 值为1表示事务是read only。                                   |
| TRX_AUTOCOMMIT_NON_LOCKING | 值为1表示事务是一个select语句，该语句没有使用for update或者shared mode锁，并且执行开启了autocommit，因此事务只包含一个语句。当`TRX_AUTOCOMMIT_NON_LOCKING`和`TRX_IS_READ_ONLY`同时为1，InnoDB 优化事务，以减少事务与更改表数据相关的开销。 |
| TRX_SCHEDULE_WEIGHT        | 由争用感知事务调度（CATS）算法分配给等待锁的事务的事务调度权重。该值相对于其他事务的值。值越大，权重越大。仅为`TRX_STATE` 值 `LOCK WAIT` 状态的事务计算。未等待锁定的事务为空值。`TRX_SCHEDULE_WEIGHT` 值与`TRX_WEIGHT` 值不同，`TRX_WEIGHT` 值是由不同的算法为不同的目的计算的。 |



## 参考资料

- 官方文档  [`15.2 InnoDB and the ACID Model`](https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html)
- 官方文档 [`15.7.2 InnoDB Transaction Model`](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)
- 官方文档 [`8.5.3 Optimizing InnoDB Read-Only Transactions`](https://dev.mysql.com/doc/refman/8.0/en/innodb-performance-ro-txn.html)
- 官方文档 [`26.4.30 The INFORMATION_SCHEMA INNODB_TRX Table`](https://dev.mysql.com/doc/refman/8.0/en/information-schema-innodb-trx-table.html)

