##  InnoDB 如何使用锁支持事务

InnoDB 使用不同的锁策略支持各个事务隔离级别。对于非常重要的关键数据操作，需要满足ACID 模型，可以指定为强一致性的默认的 可重复读 的隔离级别。在批量报表中，强一致性和可重复性结果不如最小化锁定开销那么重要，可以使用 读取提交 甚至 读取未提交 来放松一致性要求。可序列化 比 可重复读取 执行时遵循更严格的规则，主要用于特殊情况，例如 分布式事务（XA），以及解决并发和死锁问题。

下面描述了MySQL如何支持不同的事务级别，列表从最常用的级别到最少使用的级别。

### 可重复读取（REPEATABLE READ）

这是 InnoDB 的默认隔离级别。同一事务中的读一致性实现方式，采用继续读取第一次读时所建立的快照。这表示如果在同一事务中发出多个普通（非锁定）SELECT 语句，那么这些 SELECT 语句读取的结果也是一致的。

对于读锁定（`SELECT ... FOR UPDATE` 或 `FOR SHARE`），更新或删除语句，锁定方式取决于语句是否使用唯一索引作为唯一的搜索条件，还是使用区间搜索条件。

- 对于使用唯一索引作为唯一的搜索条件，InnoDB 只锁定找到的索引记录，而不锁定它前面的间隙。
- 对于其他搜索条件，InnoDB 会锁定扫描的索引范围，使用间隙锁或临键锁阻止其他会话插入该范围覆盖的间隙。

### 读取提交（ READ COMMITTED）

读一致性实现，即使在同一事务中，都设置并读取自己的新快照。

对于读锁定（`SELECT ... FOR UPDATE` 或 `FOR SHARE`），更新或删除语句，InnoDB 只锁定索引记录，而不锁定它们之前的间隔，因此允许在锁定的记录旁自由插入新记录。间隙锁定仅用于外键约束检查和重复键检查。

由于禁用了间隙锁定，可能会出现幻行问题，因为其他会话可以将新行插入间隙。

读取提交 隔离级别仅支持基于行的二进制日志记录。如果使用 读取提交 和 `binlog format=MIXED`，将自动使用基于行的日志记录。

使用 读取提交 的其他效果：

- 对于`UPDATE`或`DELETE`语句，InnoDB 只对它更新或删除的行持有锁。在 MySQL 评估 WHERE 条件之后，将释放不匹配行的记录锁。这大大降低了死锁的概率，但死锁仍然可能发生。
- 对于`UPDATE`语句，如果一行已经锁定，InnoDB 执行`semi-consistent`读取，将最新提交的版本返回 MySQL，以便MySQL 能够确定该行是否与更新的 WHERE 条件匹配。如果行匹配（必须更新），MySQL 将再次读取该行，这次InnoDB 会锁定它，或者等待对它的锁定。

参考以下示例，初始化表数据语句如下：

```sql
CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);
COMMIT;
```

表中没有索引，所以搜索和索引扫描使用隐藏的聚簇索引，来进行记录锁定，而没有使用自建索引列。

假设一个会话使用如下语句执行更新：

```sql
# Session A
START TRANSACTION;
UPDATE t SET b = 5 WHERE b = 3;
```

假设第二个会话在第一个语句之后，使用如下语句来执行更新：

```sql
# Session B
UPDATE t SET b = 4 WHERE b = 2;
```

当 InnoDB 执行每个更新时，首先为每一行获取一个独占锁，然后确定是否修改它。如果不修改对应行，则释放锁。否则，InnoDB 将保留行锁直到事务结束。这会影响事务处理，如下所示。

当使用默认的 可重复读 隔离级别时，第一个会话的更新操作，在其读取的每一行上获取独占锁（X），并且不会释放任何行：

```sql
x-lock(1,2); retain x-lock
x-lock(2,3); update(2,3) to (2,5); retain x-lock
x-lock(3,2); retain x-lock
x-lock(4,3); update(4,3) to (4,5); retain x-lock
x-lock(5,2); retain x-lock
```

第二个会话的更新操作，在尝试获取任何锁时进行阻塞（因为第一个会话的更新操作，在所有行上保留了锁），直到第一个会话的事务提交或回滚：

```sql
x-lock(1,2); block and wait for first UPDATE to commit or roll back
```

如果使用 读提交 隔离级别，则第一个会话的更新操作，将在其读取的每一行上获取独占锁（X），并释放未修改的行的 X锁：

```sql
x-lock(1,2); update(1,2) to (1,4); retain x-lock
x-lock(2,3); unlock(2,3)
x-lock(3,2); update(3,2) to (3,4); retain x-lock
x-lock(4,3); unlock(4,3)
x-lock(5,2); update(5,2) to (5,4); retain x-lock
```

但是，如果 WHERE 条件包含索引列，并且 InnoDB 使用索引，那么会使用索引列来获取和持有记录锁。在下面的示例中，第一个会话的更新操作，在 b=2 的每个索引行上获取并持有X锁。第二个会话的更新操作，在尝试获取相同记录上的X锁时阻塞，因为它也使用在 b 列上的索引。

```sql
CREATE TABLE t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2,3),(2,2,4);
COMMIT;

# Session A
START TRANSACTION;
UPDATE t SET b = 3 WHERE b = 2 AND c = 3;

# Session B
UPDATE t SET b = 4 WHERE b = 2 AND c = 4;
```

可以在启动时设置 读提交 隔离级别，也可以在运行时更改。在运行时，可以为所有会话或单个会话进行配置。

### 读取未提交（READ UNCOMMITTED）

SELECT 语句以非锁定方式执行，但可能会使用行的早期版本。因此，使用这个隔离级别，这样的读取是不一致的。这也称为脏读。否则，此隔离级别的工作方式类似于 READ COMMITTED。

### 可序列化（SERIALIZABLE）

此级别类似于可重复读取，但是如果 autocommit 被禁用，InnoDB 会隐式地将所有普通 SELECT 语句转换为`SELECT ... FOR SHARE`。如果启用了 autocommit，则 SELECT 有自己的事务。因此，已知它是只读的，如果作为一致（非锁定）读取执行，并且不需要为其他事务阻塞，则可以序列化(若要强制普通 SELECT 阻止在其他事务修改已选定行时，请禁用“自动提交”。）

## 一致性非锁定读

一致性读取意味着 InnoDB 使用多版本控制（MVCC）在某个时间点向查询提供数据库的快照。查询将看到在该时间点之前提交的事务所做的更改，而不会看到稍后或未提交的事务所做的更改。但可以看到同一事务中早期语句所做的更改。这会导致以下反常现象：如果更新表中的某些行，SELECT 可以看到更新行的最新版本，但也可能会看到任何行的旧版本。如果其他会话同时更新同一个表，这种反常现象意味着可能会看到该表的查询结果与数据库中的存储不符合。

如果事务隔离级别是 可重复读（默认级别），则同一事务中的都读取该事务中第一次读取时建立的快照。提交当前事务，再次查询就可以获取到最新的数据快照。

使用 读提交 隔离级别，事务中的每个读取都会设置并读取自己的新快照。

在 可重复读 和 读提交 隔离级别中， 一致性读取 是InnoDB处理 `SELECT` 语句的默认模式。一致性读取 不会对访问的表设置任何锁，因此其他会话可以在对表执行 一致性读取 的同时，自由修改表。

假设您在 可重复读 隔离级别下（默认级别）。当发出一致性读取（即普通的 SELECT 语句）请求时，InnoDB 会给事务一个时间点，当前事务中，相同的查询看到的都是这个时间点对应的数据库快照。如果另一个事务删除一行并在分配了时间点之后提交，则不会认为该行已被删除。插入和更新的处理方式类似。

> **备注**
>
> 数据库的快照应用于事务中的 SELECT 语句，而不一定适用于 DML 语句。如果插入或修改某些行并提交该事务，则从另一个 可重复读 事务发出的 DELETE 或 UPDATE 语句可能会影响那些刚刚提交的行，即使会话无法查询它们。如果事务确实更新或删除由其他事务提交的行，那么这些更改对当前事务可见。例如，您可能遇到如下情况：
>
> ```sql
> SELECT COUNT(c1) FROM t1 WHERE c1 = 'xyz';
> -- 返回 0: 没有匹配数据。
> DELETE FROM t1 WHERE c1 = 'xyz';
> -- 删除其他事务最近提交的几行。
> 
> SELECT COUNT(c2) FROM t1 WHERE c2 = 'abc';
> -- 返回 0: 没有匹配数据。
> UPDATE t1 SET c2 = 'cba' WHERE c2 = 'abc';
> -- 更新了 10 行: 其他的事务刚提交了10行 c2 = 'abc' 的数据。
> SELECT COUNT(c2) FROM t1 WHERE c2 = 'cba';
> -- 返回 10: 这个事务可以看到它刚更新的10行数据。
> ```

您可以提交事务，然后执行另一个 `SELECT` 或 `START TRANSACTION WITH CONSISTENT SNAPSHOT`语句，来更新当前会话的时间点快照。

这称为多版本并发控制（`multi-versioned concurrency control`，简称 MVCC）。

在下面的示例中，会话 A 仅在 B 提交了 insert 并且 A 也提交了 insert 后，才能看到 B 插入的行，因为时间点更新到 B 提交之后。

```sql
             Session A              Session B

           SET autocommit=0;      SET autocommit=0;
time
|          SELECT * FROM t;
|          empty set
|                                 INSERT INTO t VALUES (1, 2);
|
v          SELECT * FROM t;
           empty set
                                  COMMIT;

           SELECT * FROM t;
           empty set

           COMMIT;

           SELECT * FROM t;
           ---------------------
           |    1    |    2    |
           ---------------------
```

如果要查看数据库的“最新”状态，请使用 读提交 隔离级别或锁定读取：

```sql
SELECT * FROM t FOR SHARE;
```

使用 读提交 隔离级别，事务中的读取都会设置并读取自己的新快照。对于`SELECT ... FOR SHARE`，将执行锁定读取：SELECT 阻塞，直到包含最新行的事务结束。

一致性读取  不适用于某些 DDL 语句：

- 一致性读取 不能和`DROP TABLE` 语句工作，因为 MySQL 无法使用已删除的表，InnoDB 会破坏该表。
- 一致性读取 不适用于创建表的临时副本，并在生成临时副本时删除原表的`ALTER TABLE`操作。在事务中再次发出一致读取时，新表中的行不可见，因为在获取事务快照时，这些行不存在。在这种情况下，事务返回一个错误：ER_TABLE_DEF_CHANGED（表定义已更改，请重试事务）。

对于 SELECT IN 子句，读方式各不相同，例如：`INSERT INTO ... SELECT`，`UPDATE ... (SELECT)`，`CREATE TABLE ... SELECT` 这些没有指定 `FOR UPDATE` 或 `FOR SHARE` 锁定格式的。

- 默认情况下，InnoDB 对这些语句使用更强的锁，SELECT 部分的行为类似于  读提交 隔离级别下的处理方式，其中每个一致性读取（即使在同一事务中）都设置并读取自己的新快照。
- 要在这种情况下执行非锁定读取，请将事务的隔离级别设置为 读未提交 或 读提交，以避免对选定表中读取的数据行设置锁。

## 读锁定

如果查询数据，然后在同一事务中插入或更新相关数据，则常规SELECT语句不能提供足够的保护。其他事务可以更新或删除刚才查询的相同行。InnoDB支持两种提供更强安全性的锁定读取：

- `SELECT ... FOR SHARE`：在读取的行设置共享锁。其他会话可以读取行，但在事务提交之前不能修改它们。如果这些行中的任何一行被另一个尚未提交的事务更改，则查询将等待该事务结束，然后使用最新的值。

  > `SELECT ... FOR SHARE` 是对`SELECT ... LOCK IN SHARE MODE`的替代。但 `SELECT ... LOCK IN SHARE MODE` 仍然向后保持兼容，它们是等价的。然后，`FOR SHARE` 支持 `table_name`， `NOWAIT`，和 `SKIP LOCKED` 选项。

  在 MySQL 8.0.22 之前，`SELECT ... FOR SHARE` 需要 `SELECT` 权限和至少一个`DELETE`、`LOCK TABLES` 或 `UPDATE` 权限。在 MySQL 8.0.22 中，只需要 `SELECT` 权限。

  在 MySQL 8.0.22 中, `SELECT ... FOR SHARE` 语句不需要 MYSQL 授权表上的读锁。

- `SELECT ... FOR UPDATE`：对于搜索遇到的索引记录，锁定数据行和任何关联的索引项，就像为这些行发出 `UPDATE` 语句一样。其他事务不能更新这些行，不能执行`SELECT ... FOR SHARE`，或在某些事务隔离级别下进行数据读取。一致性读 忽略在“读取”视图中的记录上设置的任何锁（记录的旧版本不能被锁定，它们是通过撤消日志，在记录的内存副本上重建的）。

  `SELECT ... FOR UPDATE` 需要 `SELECT` 权限和至少一个`DELETE`、`LOCK TABLES` 或 `UPDATE` 权限。

这些语句在处理树形或图形结构化数据时非常有用，无论是在单个表中还是多个表。可以在一个位置遍历边或树分支，完成时不仅可以保证数据的正确性，并且可以更改任意值。

提交或回滚事务时，将释放由`FOR SHARE`和`FOR UPDATE`查询设置的所有锁。

> 锁定读取 仅在禁用 autocommit 时才生效（通过使用`START TRANSACTION`开始事务或将 autocommit 设置为 0）。

除非子查询中也指定了读锁定，否则外部语句中的读锁定不会锁定子查询中的行。例如，以下语句不会锁定表 t2 中的行。

```sql
SELECT * FROM t1 WHERE c1 = (SELECT c1 FROM t2) FOR UPDATE;
```

要锁定表t2中的行，要向子查询也添加一个读锁定：

```sql
SELECT * FROM t1 WHERE c1 = (SELECT c1 FROM t2 FOR UPDATE) FOR UPDATE;
```

**读锁定示例**

假设要将新行插入`child`表，并确保子行在`parent`表中有父行。应用程序代码可以确保整个操作序列中的引用完整性。

首先，使用一致性读取来查询`parent`表并验证父行是否存在。您能安全地将子行插入`child`表吗？不可以，因为其他会话可能会在`SELECT`和`INSERT`之间删除父行，而本会话对此一无所知。

要避免此潜在问题，需要使用`SELECT ... FOR SHARE`：

```sql
SELECT * FROM parent WHERE NAME = 'Jones' FOR SHARE;
```

在`FOR SHARE`查询返回父`Jones`之后，可以安全地将子记录添加到`child`表并提交事务。任何想在`parent`表获取相关行独占锁的事务都会阻塞，直到本事务完成，也就是说，直到所有表中的数据处于一致状态。

另一个示例，考虑表`child_codes`中的整数`counter_field`字段，用于分配唯一标识给添加到表`child`的每条数据。不要使用一致性读取或共享模式读取来读取`counter_field`的当前值，因为数据库的多个用户都可以看到`counter_field`的相同值，如果两个事务尝试向子表中添加具有相同标识符的行，则会发生重复键错误。

这种场景下，`FOR SHARE`并不是一个好的解决方案，因为如果两个用户同时读取`counter_field`，那么在尝试更新`counter_field`时，至少其中一个用户会以触发死锁。

要实现对`counter_field`的读取和递增，首先使用`FOR UPDATE`对计数器执行锁定读取，然后递增`counter_field`。例如：

```sql
SELECT counter_field FROM child_codes FOR UPDATE;
UPDATE child_codes SET counter_field = counter_field + 1;
```

`SELECT ... FOR UPDATE`读取最新可用数据，并在读取的每一行设置独占锁。因此，它与 `UPDATE` 语句设置了相同的锁在数据行上。

前面只是一个`SELECT ... FOR UPDATE`的使用示例。在MySQL中，生成唯一标识符的特定任务实际上只需对表进行一次访问：

```sql
UPDATE child_codes SET counter_field = LAST_INSERT_ID(counter_field + 1);
SELECT LAST_INSERT_ID();
```

`SELECT`语句仅检索标识符信息（特定于当前连接）。不访问任何表。

**用`NOWAIT`和`SKIP`锁定读并发**

如果某行被某个事务锁定，则`SELECT ... FOR UPDATE`或`SELECT ... FOR SHARE`对于请求相同锁定行的事务，需要阻塞等待事务释放行锁。为了防止事务更新或删除其他事务查询并更新的行。但如果希望查询时，请求的行被锁定立即返回，或者可以接受排除锁定数据行的结果集，则无需等待行锁释放。

为了避免等待其他事务释放行锁，可以使用`NOWAIT`和`SKIP LOCKED`参数与`SELECT ... FOR UPDATE`或`SELECT ... FOR SHARE`进行查询。

`NOWAIT`：使用`NOWAIT`的锁定读不等待获取行锁，查询将立即执行，如果请求的行被锁定，查询将失败并返回错误。

`SKIP LOCKED`：使用`SKIP LOCKED`的锁定读不等待获取行锁，查询立即执行，返回去除锁定数据行的结果集。

> 跳过锁定行的查询返回不一致的数据视图。因此，`SKIP LOCKED`不适用于事务性工作。但当多个会话访问同一个类似队列的表时，它可以用来避免锁争用。

`NOWAIT`和`SKIP LOCKED`仅适用于行级锁。

使用`NOWAIT`或`SKIP LOCKED`对使用语句进行复制的操作，是不安全。

`NOWAIT`和`SKIP LOCKED`示例。会话1启动一个事务，该事务采用行锁。会话2尝试使用`NOWAIT`选项对同一记录进行锁定读取。因为请求的行被会话1锁定，所以锁定读取会立即返回错误。在会话3中，带`SKIP LOCKED`的读锁定返回请求的行，排除了会话1锁定的行。

```sql
# Session 1:
mysql> CREATE TABLE t (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
mysql> INSERT INTO t (i) VALUES(1),(2),(3);
mysql> START TRANSACTION;
mysql> SELECT * FROM t WHERE i = 2 FOR UPDATE;
+---+
| i |
+---+
| 2 |
+---+

# Session 2:
mysql> START TRANSACTION;
mysql> SELECT * FROM t WHERE i = 2 FOR UPDATE NOWAIT;
ERROR 3572 (HY000): Do not wait for lock.

# Session 3:
mysql> START TRANSACTION;
mysql> SELECT * FROM t FOR UPDATE SKIP LOCKED;
+---+
| i |
+---+
| 1 |
| 3 |
+---+
```

## 参考资料

- 官方文档 [15.7.2 InnoDB Transaction Model](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)

