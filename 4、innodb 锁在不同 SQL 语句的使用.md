## InnoDB 锁在不同 SQL 语句的使用

锁定读取、更新或删除SQL语句通常会设置扫描的每个索引的记录锁。不关注是条件。InnoDB不记录条件范围，只知道扫描了哪些索引区间。锁通常是临键锁，它也会阻止在记录前插入到“间隙”中。间隙锁定可以显式禁用，这会导致不采用临键锁。事务隔离级别也会影响设置哪些锁。

如果在搜索中使用二级索引，并且要设置的索引记录锁是独占的，InnoDB还会检索相应的聚簇索引记录并对其设置锁。

如果没有适合的索引，MySQL必须扫描整个表来处理语句，则表的每一行都会被锁定，这反过来会阻止其他用户对表的所有插入。创建好的索引非常重要，这样查询就只会扫描那些需要的行。

InnoDB设置特定类型的锁，如下所示：

### SELECT 

`SELECT ... FROM`：一致性读取，读取数据库的快照，不设置锁。除非事务隔离级别设置为`SERIALIZABLE`时，在遇到的索引记录上设置共享临键锁。对于使用唯一索引搜索唯一行的语句，只需要索引记录锁。

`SELECT ... FOR UPDATE` 和 `SELECT ... FOR SHARE`：使用唯一索引的语句获取扫描行锁，然后释放不符合条件的行锁（即不满足`WHERE`的条件）。但在某些特殊情况下，行不会立即解锁，因为查询执行期间结果行与其原始数据之间的关系丢失。例如，在联合查询中，表扫描（并锁定）的行可能会在判断它们是否满足条件之前插入到临时表中，在这种情况下，临时表中的行与原表中的行之间的关系将丢失，并且在查询执行结束之前，原表中的行不会被解锁。

对于读锁定（`SELECT ... FOR UPDATE` 或 `FOR SHARE`）、更新或删除语句，所获取的锁取决于语句是否使用具有唯一索引的唯一搜索条件。

- 对于使用唯一索引搜索唯一行的语句，只锁定索引记录。
- 对于其他搜索条件，以及对于非唯一索引，InnoDB将扫描的索引范围锁定，使用间隙锁或临键锁阻止其他会话插入到范围所覆盖的间隙中。

对于搜索扫描到的索引记录，`SELECT ... FOR UPDATE` 阻止其他会话执行`SELECT ... FOR SHARE`或某些事务隔离级别下的读取操作。一致性读 忽略记录上的所有锁。

### UPDATE

`UPDATE ... WHERE ...` 在搜索扫描的记录上设置独占的临键锁。对于使用唯一索引搜索唯一行的，只需要索引记录锁。

`UPDATE`修改聚簇索引记录时，会隐式对受影响的二级索引记录加锁。更新操作在插入二级索引之前执行重复检查时，和在插入新的二级索引记录时，依然对受影响的二级索引记录使用共享锁。

### DELETE 

`DELETE FROM ... WHERE ...`在搜索扫描的记录上设置独占的临键锁。对于使用唯一索引搜索唯一行的，只需要索引记录锁。

### INSERT

`INSERT`在插入的行上设置独占锁。此锁是索引记录锁，不是临键锁（没有间隙锁），不阻止其他会话在行之前的间隙执行插入操作。

在插入行之前，设置一种名为插入意图间隙锁的间隙锁类型。此锁表示有意插入，这样一来，如果插入到同一索引间隙中的多个事务不在间隙中的同一位置插入，则不必等待对方执行完成。假设有索引记录值为 4 和 7。两个独立事务分别想插入值 5 和 6 ，在获得插入行的独占记录锁之前，用插入意图锁锁定 4 和 7 之间的间隙，因为插入值不冲突，因此不会互相阻塞。

如果出现重复键错误（duplicate-key），则在重复索引记录上设置共享锁。如果有多个会话试图插入同一行，而另一个会话已经具有独占锁，则使用共享锁可能会导致死锁。如果持有独占锁的会话删除该行，也可能导致死锁。假设InnoDB表t1具有以下结构：

```sql
CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
-- TODO 为什么重复键错误，要在索引记录上设置共享锁？
```

假设三个会话按顺序执行以下操作：

```sql
-- session 1
START TRANSACTION;
INSERT INTO t1 VALUES(1);
-- session 2
START TRANSACTION;
INSERT INTO t1 VALUES(1);
-- session 3
START TRANSACTION;
INSERT INTO t1 VALUES(1);
-- session 1
ROLLBACK;
```

会话1的第一个操作获取该行的独占锁。会话2和会话3的操作都会导致一个重复键错误，并且它们都请求为该行获取一个共享锁。当会话1回滚时，它释放其在行上的独占锁，并处理会话2和3 的共享锁请求。此时，会话2和会话3死锁：由于另一个会话持有共享锁，因此两个会话都不能获取该行的独占锁。

如果表中已包含键值为1的行，并且三个会话按顺序执行以下操作，则会出现类似情况：

```sql
-- session 1
START TRANSACTION;
DELETE FROM t1 WHERE i = 1;
-- session 2
START TRANSACTION;
INSERT INTO t1 VALUES(1);
-- session 3
START TRANSACTION;
INSERT INTO t1 VALUES(1);
-- session 1
ROLLBACK;
```

会话1的第一个操作获取该行的独占锁。会话2和会话3的操作都会导致一个重复键错误，并且它们都请求为行提供一个共享锁。当会话1提交时，它会释放其在行上的独占锁，并处理会话2和3 的共享锁请求。此时，会话2和会话3死锁：由于另一个会话持有共享锁，因此两个会话都不能获取该行的独占锁。

### 复合语句

`INSERT ... ON DUPLICATE KEY UPDATE` 与 `INSERT`不同之处在于，当出现重复键错误时，将在要更新的行上放置一个独占锁而不是共享锁。对重复的主键值采用独占记录锁。对重复的唯一键值采用独占的临键锁。

如果唯一键上没有冲突，则`REPLACE`就像`INSERT`一样完成。否则，将在要替换的行上放置一个独占的临键锁。

`INSERT INTO T SELECT ... FROM S WHERE ...` 在插入到T中的每一行上设置独占索引记录锁（不带间隙锁）。如果事务隔离级别为 可重复读，则 InnoDB 会以一致性读（无锁）的方式对S进行搜索。否则，InnoDB会对S中的行设置共享的临键锁。在后一种情况下，InnoDB必须设置锁：在使用基于语句的二进制日志进行回滚恢复的过程中，每个SQL语句的执行方式必须与最初完全相同。

`CREATE TABLE ... SELECT ...` 同  `INSERT ... SELECT`。

`REPLACE INTO t SELECT ... FROM s WHERE ...`或`UPDATE t ... WHERE col IN (SELECT ... FROM s ...)` 中的`SELECT`，InnoDB 在表 s 的行上设置共享的临键锁。 

InnoDB 化的`AUTO_INCREMENT`列时，在与`AUTO_INCREMENT`列关联的索引的末尾设置一个独占锁。

当`innodb_autoinc_lock_mode=0`时，innodb 使用一种特殊的`AUTO-INC`表锁模式，在访问自动递增计数器时，锁被获取并持续到当前SQL语句的结尾（而不是整个事务的结尾）。当`AUTO-INC`表锁被持有时，其他客户端无法插入到表中。对于`innodb_autoinc_lock_mode=1`的“批量插入”，行为相同。`innodb_autoinc_lock_mode=2`模式下，不采用`AUTO-INC`锁。更多信息，请查看官方文档 [15.6.1.6 InnoDB 自增处理](https://dev.mysql.com/doc/refman/8.0/en/innodb-auto-increment-handling.html)。

InnoDB 获取先前初始化的自动增量列的值，不设置任何锁。

如果在表上定义了`FOREIGN KEY`约束，则任何需要检查约束条件的`insert`、`update`或`delete`都会检查约束的记录，并设置共享记录锁。约束失败也设置。

`LOCK TABLES`设置表锁，是`MySQL`层设置的锁。如果`innodb_table_locks = 1`（默认值）和`autocommit=0`，InnoDB 知道表锁，InnoDB 上层的 MySQL 知道行级锁。

否则，InnoDB的自动死锁检测无法检测到涉及此类表锁的死锁，另外由于在这种情况下，MySQL层不知道行级锁，所以可以在另一个会话当前具有行级锁的表上获得表锁，但这不会危及事务完整性，如 `15.7.5.2 死锁检测` 中所述。

如果`innodb_table_locks=1`（默认值），`LOCK TABLES`在每个表上获取两个锁。除了MySQL层的表锁之外，它还获得了一个InnoDB表锁。为避免获取InnoDB表锁，请将innodb_table_locks=0，这种情况下，则即使表的某些记录被其他事务锁定，锁表也会完成。

在MySQL 8.0中，`innodb_table_locks=0`对使用 `LOCK TABLES ... WRITE` 显式锁表没有影响。但对`LOCK TABLES ... WRITE` 隐式锁表（例如，通过触发器）的读和写 或  `LOCK TABLES ... READ` 显式共享读锁表。

事务提交或中止时，事务持有的所有InnoDB锁都会被释放。因此，在`autocommit=1`模式下调用InnoDB表上的`LOCK TABLES`没有多大意义，因为获取的InnoDB表锁将立即被释放。

不能在事务中间锁定其他表，因为`LOCK TABLES`执行隐式`COMMIT`和`UNLOCK TABLES`。

## 参考资料

- 官方文档 [15.7.3 Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/8.0/en/innodb-locks-set.html)

- 官方文档 [13.3 Transactional and Locking Statements](https://dev.mysql.com/doc/refman/8.0/en/sql-transactional-statements.html)


