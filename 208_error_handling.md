错误处理
========
WiredTiger在操作成功时返回0，在出错是返回非零值。错误码可能是正数或者负数：正数是类POSIX系统的标准错误码（例如，EINVAL或EBUSY），负数是WiredTiger特定的（例如，WT_ROLLBACK）。

WiredTiger的错误码在-31800到-31999之间。

下面是部分WiredTiger的错误码：

**WT_ROLLBACK**

这个错误发生在由于并发操作冲突导致操作不能完成时。操作可以重试，如果事务还在进行，那么应该将事务回滚并在一个新事务中重试。

**WT_DUPLICATE_KEY**

这个错误发生在应用程序试图插入的记录与已存在的记录有相同的键，并且没有为`WT_SESSION::open_cursor`配置'overwrite'。

**WT_ERROR**

这个错误发生在错误没有被特定的错误码覆盖时。

**WT_NOTFOUND**

这个错误表示操作没有找到要返回的值。这包括游标查找和没有记录能够匹配游标的查找键的其它操作，像`WT_CURSOR::update`和`WT_CURSOR::remove`。

**WT_PANIC**

这个错误表示需要应用程序退出重启的底层错误。当WiredTiger的接口返回WT_PANIC时应用程序可以立即退出，不需要调用其它WiredTiger接口。

**WT_RUN_RECOVERY**

这个错误发生在`wiredtiger_open`被配置成当数据库需要恢复时返回错误。

`wiredtiger_strerror`函数返回与任何WiredTiger、ISO C99或者POSIX 1003.1-2001函数相关的标准消息：
```c
const char *key = "non-existent key";
cursor->set_key(cursor, key);
if ((ret = cursor->remove(cursor)) != 0) {
    fprintf(stderr, "cursor.remove: %s\n", wiredtiger_stderror(ret));
    return (ret);
}
```

更复杂的错误处理可以通过传递一个`WT_EVENT_HANDLER`的实现给`wiredtiger_open`或`WT_CONNECTION::open_session`来配置。
