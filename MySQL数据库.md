# MySQL数据库

#### 锁

在Innodb中，同时支持表锁和行锁，表锁和行锁均有两种：**共享锁**和**排他锁**，以行锁为例：

​	**共享锁（shared lock）**：

​		共享锁允许获取到锁的事务对行数据进行读取。

​	**排他锁（exclusive lock）**：

​		排他锁允许获取到锁的事务对行数据进行删除和更新。

共享锁和排他锁的互斥关系如表所示：

|      | S    | X    |
| ---- | ---- | ---- |
| S    | 共存 | 互斥 |
| X    | 互斥 | 互斥 |

**意向锁：**意向锁是一种加在表上面的锁，表明事务接下来想要获取的行锁是哪一种（排他锁还是共享锁）。意向锁有两种：意向共享锁、意向排他锁。

​	**意向共享锁（IS）**：表明事务准备获取个别行的共享锁。

​	**意向排他锁（IX）**：表明事务准备获取个别行的排他锁。

意向锁存在的目的是为了解决表锁和行锁之间存在的冲突。考虑这种情况：事务A锁住了表中的一行，让这一行只能读，不能写，之后事务B申请整个表的写锁，如果B申请成功，那么理论上它可以需修改任意一行，显然这和A的行锁是矛盾的，因此数据库需要判断该冲突，

第一步：判断表是否被其他事务用表锁锁住

第二步：判断表中每一行是否被行锁锁住

第一步很好判断，第二步效率很低，因此需要意向锁，事务在获取行锁之前需要获取表上的意向锁，上述的判断过程变成了如下所示：

第一步：不变

第二步：发现表上有意向锁，说明表中有些行被共享行锁锁住了，因此事务B申请表的写锁会被阻塞。

表锁的兼容性如下表所示：

|      | X    | IX   | S    | IS   |
| ---- | ---- | ---- | ---- | ---- |
| X    | 冲突 | 冲突 | 冲突 | 冲突 |
| IX   | 冲突 | 兼容 | 冲突 | 兼容 |
| S    | 冲突 | 冲突 | 兼容 | 兼容 |
| IS   | 冲突 | 兼容 | 兼容 | 兼容 |

**Record Lock**：recordlock是索引上面的锁，例如，select c1 from t where c1 = 10 for update；该语句阻止其他事务对c1 = 10 这条记录进行增、删、改的操作。

**Gap Locks（间隙锁）**：间隙锁锁住了索引之间的间隙，或者是锁住了第一个值之前的间隙或者最后一个值后面的间隙。例如：select c1 from t where c1 between 10 and 20 for update; 如果此时想插入c1 = 15，会被阻塞，不管有没有15这个值，因为间隙锁锁住了10-20之间的间隙。**间隙锁是共存的**（？），一个事务获取到的间隙锁，另一个事务也可以获取完全一样的间隙锁。

**Next-Key Locks**：next-key lock是索引上record lock和索引前gap lock的混合体。Innodb的行级锁就是以这样的方式体现。

**Insert Intention Locks**：

**AUTO-INC Locks**：

**不同sql语句使用到的锁：**

具体参考：

[官方文档]: https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html

1. select...from：一致性读，读取数据库的一个快照，不会设置锁，除非数据库的隔离级别是SERIALIZABLE。对于SERIALIZABLE隔离级别，在搜索过程中遇到的每一个记录，就会加上一个共享间隙锁。
2. select...for update：在搜索过程中遇到的每一个记录，就会加上一个排他间隙锁。
3. select...lock in share mode：在搜索过程中遇到的每一个一个记录，都会加上一个共享间隙锁。
4. update...where...：在搜索过程中遇到的每一个记录，就会加上一个排他间隙锁。
5. delete from...where...：在搜索过程中遇到的每一个记录，就会加上一个排他间隙锁。
6. insert：在被插入的行中加上排他锁，该排他锁不是间隙锁，而实index-record lock。

mvcc可以避免幻读吗？这要看幻读怎么理解，如果单纯的读取，MVCC可以避免，具体如下：

创建空表table t:

执行过程：

| 事务A                                 | 事务B                              |
| ------------------------------------- | ---------------------------------- |
| begin                                 | begin                              |
| select * from t;(empty)               |                                    |
|                                       | insert into t values (3,3);commit; |
| select * from t; (empyt)              |                                    |
| update t set a = 10;（1 row effects） |                                    |
| commit;                               |                                    |

也就是说，事务B插入数据提交之后，事务A select的返回结果跟第一次select的返回结果一致，但是事务A更新的时候，会发现一行受到影响，这样不知道算不算是一种幻读。此处引用官方的说法：

> Note
>
> The snapshot of the database state applies to [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) statements within a transaction, not necessarily to [DML](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_dml) statements. If you insert or modify some rows and then commit that transaction, a [`DELETE`](https://dev.mysql.com/doc/refman/5.7/en/delete.html) or [`UPDATE`](https://dev.mysql.com/doc/refman/5.7/en/update.html) statement issued from another concurrent `REPEATABLE READ` transaction could affect those just-committed rows, even though the session could not query them. If a transaction does update or delete rows committed by a different transaction, those changes do become visible to the current transaction. For example, you might encounter a situation like the following:
>
> ```mysql
> SELECT COUNT(c1) FROM t1 WHERE c1 = 'xyz';
> -- Returns 0: no rows match.
> DELETE FROM t1 WHERE c1 = 'xyz';
> -- Deletes several rows recently committed by other transaction.
> 
> SELECT COUNT(c2) FROM t1 WHERE c2 = 'abc';
> -- Returns 0: no rows match.
> UPDATE t1 SET c2 = 'cba' WHERE c2 = 'abc';
> -- Affects 10 rows: another txn just committed 10 rows with 'abc' values.
> SELECT COUNT(c2) FROM t1 WHERE c2 = 'cba';
> -- Returns 10: this txn can now see the rows it just updated.
> ```

如果要真正解决幻读，那么需要使用locking read，也就是再select后面加上 for update 或者 lock in share mode。此时的查询会给扫描到的记录加上netx-key record，防止其他事务在这个间隙中update、delete、insert进去。

#### 事务

原子性（Atomicity）：一个事务必须被视为一个不可分割的最小工作但愿，整个事务中的所有操作要么全部提交成功，要么全部失败回滚，对于一个事务来说，不可能只执行其中的一部分。

一致性（Consistency）：数据库总是从一个一致性的状态转到另一个一致性的状态。

隔离性（Isolation）：一个事务所作的修改在最终提交之前，对其他事务是不可见的。

持久性（durability）：一旦事务提交，则其所作的修改就会永久保存在数据库中。

#### 隔离级别

**READ UNCOMMITTED（读未提交）**

​	在READ UNCOMMITTED级别下，事务中的修改，即使没有提交，对其他事务也都是可见的。事务可以读取未提交的数据，也被称为脏读（Dirty Read）。从性能上来说，READ UNCOMMITTED不会比其他隔离级别好太多，但是会造成更多的问题，因此一般很少使用。

**READ COMMITTED（读已提交）**

​	事务从开始到提交之前，所作的任何修改对其他事务都是不可见的。存在的问题为不可重复读（nonrepeatable read）,一个事务中两次执行同样的查询，可能会得到不一样的结果。

**REPETABLE READ（可重复读）（MySQL默认隔离级别）**

​	在同一个事务中，多次读取同样的记录结果是一样的。存在幻读问题，即当某个事务在读取某个范围内的记录时，另一个事务又在该范围内插入了新纪录，当之前的事务再次读取该范围的记录时，会产生幻行。InnoDB存储引擎通过MVCC（多版本并发控制）解决了幻读问题。

**SERIALIZABLE（可串行化）**

​	最高隔离级别，强制数据串行执行，避免了幻读的问题。

| 隔离级别         | 脏读可能性 | 不可重复读可能性 | 幻读可能性 | 加锁读 |
| ---------------- | ---------- | ---------------- | ---------- | ------ |
| READ UNCOMMITTED | Yes        | Yes              | Yes        | NO     |
| READ COMMITTED   | No         | Yes              | Yes        | NO     |
| REPETABLE READ   | NO         | NO               | Yes        | NO     |
| SERIALIZABLE     | No         | No               | No         | Yes    |

#### 死锁：

两个或多个事务在同一资源上相互占用，并请求锁定对方资源，而导致恶性循环的现象。

解决方法：Innodb通过回滚持有最少行级排他锁的事务，便可以打破死锁。

#### 多版本并发控制（MVCC）

start_version <= transaction_version <= delete_version

