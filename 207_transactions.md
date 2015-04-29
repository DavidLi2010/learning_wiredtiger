事务
====

**ACID特性**

事务为多线程并发操作数据提供了强大的抽象，因为它具有以下特性：
- 原子性：事务的所有操作或者都完成了或者一个都没完成。
- 一致性：如果把事务独立看待时每个事务保持了一些特性，那么事务并发执行后组合起来的效果也保持了相同的特性。
- 隔离性：开发人员可以推断事务好像单线程运行的。
- 持久性：一旦事务提交了，那么它的更新就不会丢失。

WiredTiger支持事务，对ACID特性有如下警告：
- 支持的最高隔离级别是快照(snapshot)。
- 由检查点(checkpoint)和日志(logging)联合起来保证事务更新的持久化。
- 为了效率，每个事务的未提交更改必须在内存中完成。WiredTiger直到事务提交时才会写日志。

**事务API**

在WiredTiger中，事务操作是WT_SESSION类的方法。

应用程序通过调用`WT_SESSION::begin_transaction`开启事务，使用该WT_SESSION句柄的随后执行的操作--包括该WT_SESSION句柄打开的任何游标（无论是在`WT_SESSION::begin_transaction`之前打开的还是之后打开的）--都是事务的一部分，通过调用`WT_SESSION::commit_transaction`来提交，或者调用`WT_SESSION::rollback_transaction`来放弃。

无论什么原因，如果`WT_SESSION::commit_transaction`返回错误，事务会被回滚而不提交。

当使用事务的时候，数据操作可能会发生冲突而导致`WT_ROLLBACK`错误。如果发生了这个错误，事务应该通过`WT_SESSION::rollback_transaction`回滚，然后重试操作。

`WT_SESSION::rollback_transaction`方法隐式的复位了会话上的所有游标--就像调用了`WT_CURSOR::reset`一样--丢弃了所有游标的位置以及键和值。

```c
/*
 * Cursors may be opened before or after the transaction begins, and in
 * either case, subsequent operations are included in the transaction.
 * Opening cursors before the transaction begins allows applications to
 * cache cursors and use them for multiple operations.
 */
ret = session->open_cursor(session, "table:mytable", NULL, NULL, &cursor);
ret = session->begin_transaction(session, NULL);

cursor->set_key(cursor, "key");
cursor->set_value(cursor, "value");
switch(ret = cursor->update(cursor)) {
case 0:              /* Update success */
    ret = session->commit_transaction(session, NULL);
    /*
     * If commit_transaction succeeds, cursors remain positioned; if
     * commit_transaction fails, the transaction was rolled-back and
     * and all cursors are reset.
     */
    break;
case WT_ROLLBACK:    /* Update conflict */
default:             /* Other error */
    break;
 }

 /*
  * Cursors remain open and may be used for multiple transactions.
  */
```

**隐式事务**

如果在会话中没有显式的事务活动时，游标的读操作使用会话的隔离级别。通过为`WT_CONNECTION::open_session`设isolation键来设置会话的隔离级别，更新操作会在执行成功返回前自动提交。

任何由多个相关的更新组成的操作都应该显式地包含在事务中以确保更新操作的原子性。

如果隐式事务成功提交，WT_SESSION上的游标会保存位置。如果隐式事务失败了，WT_SESSION上的所有的游标都会被复位。

**并发控制**

WiredTiger使用优化的并发控制算法，避免了集中式的锁管理器的瓶颈，确保事务操作不会阻塞：读不阻塞写，反之亦然。

更进一步，写不阻塞写，但是并发事务更新同一个值会导致WT_ROLLBACK失败。一些应用程序会从应用程序级别的同步中受益，这些应用避免重复尝试回滚和更新同一个值。

如果某些资源返回尝试都不能被分配，事务中的操作同样会因WT_ROLLBACK错误而失败。例如，如果因为缓存不够大，达不到更新的要求，而无法满足事务的读取，操作会失败并返回WT_ROLLBACK。

**隔离级别**

WiredTiger支持读未提交、读提交和快照的隔离级别，默认的隔离级别是提交读。

- 读未提交：事务可以看到其它事务还没有提交的修改。脏读、不可重复读和幻读都有可能。
- 读提交：事务不能看到其它事务还没有提交的修改。不会出现脏读，不可重复读和幻读都有可能。在读提交的事务中，当游标没有定位时，提交的修改对其它并发的事务变得可见。
- 快照：事务读取在事务开始前记录的已提交版本。不会出现脏读和不可重复读，幻读有可能。

快照隔离级别有很强的保证，但是不等于事务单线程执行--这被称为序列化的隔离级别。并发事务T1和T2运行在快照隔离级别下，它们都有可能提交，并且在T1读T2写或者T1写T2读重叠时产生一种状态，这种状态在（T2在T1之后）和（T1在T2之后）时都不会发生。

事务的隔离级别可以在每个事务上配置：
```c
/* A single transaction configured for snapshot isolation. */
ret = session->open_cursor(session, "table:mytable", NULL, NULL, &cursor);
ret = session->begin_transaction(session, "isolation=snapshot");
cursor->set_key(cursor, "some-key");
cursor->set_value(cursor, "some-value");
ret = cursor->update(cursor);
ret = session->commit_transaction(session, NULL);
```

另外，可以在会话上配置和重新配置默认的隔离级别：
```c
/* Open a session configured for read-uncommitted isolation. */
ret = conn->open_session(conn, NULL, "isolation=read_uncommitted", &session);
```
```c
/* Re-configure a session for snapshot isolation */
ret = session->reconfigure(session, "isolation=snapshot");
```
