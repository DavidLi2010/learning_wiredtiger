数据源
======
WiredTiger提供了访问各种数据源的能力。在最底层，数据可以采用树结构存储在文件中。在文件之上，关系模式支持表、索引和列组。其它的数据源包括LSM树和统计信息，应用程序可以通过实现`WT_DATA_SOURCE`接口来扩展支持的类型。

在所有数据源上的常见操作都是通过WT_CURSOR句柄来执行的。查看[游标操作](204_cursor_operations.md)了解如何使用游标。

###内置数据源

以下是内置的基本游标类型：

|URI|类型|说明|
|---|----|----|
|table:&lt;table name>[&lt;projection>]|表游标|table key, table value(s) with optional projection of columns|
|colgroup:&lt;table name>:&lt;column group name>|列组游标|table key, column group value(s)|
|index:&lt;table name>:&lt;index name>[&lt;projection>]|索引游标|key=index key, value=table value(s) with optional projection of columns|

有一些管理任务可以使用以下特殊的游标类型来完成，这些游标能够访问由WiredTiger管理的数据：

|URI|类型|说明|
|---|----|----|
|backup:|备份游标|key=string, see Backups for details|
|log:|日志游标|	key=(long fileID, long offset, int seqno), value=(uint64_t txnid, uint32_t rectype, uint32_t optype, uint32_t fileid, WT_ITEM key, WT_ITEM value), see Log cursors for details|
|metadata:|元数据游标|key=string, value=string, see Reading WiredTiger Metadata for details|
|statistics:[&lt;data source URI>]|数据库或数据源统计信息游标|key=int id, value=(string description, string value, uint64_t value), see Statistics Data for details|

更高级的应用程序可能会打开下面的低级游标类型：

|URI|类型|说明|
|---|----|----|
|file:&lt;file name>|文件游标|file key, file value|
|lsm:&lt;name>|LSM游标（key=LSM key, value=LSM value）|LSM key, LSM value|

**Raw文件**

通过在打开游标的时候使用"file:"和底层文件的名字，WiredTiger的模式层可以被绕过。这在查看列组或索引的内容而不读取表中的列时是有用的。

例如，如果索引与它的主数据不一致时，文件游标可以不出错的从索引中读取数据（虽然一些返回的键可能在主数据中不存在）。

**表索引数据**

当创建表的索引后，无论表何时被更新，记录都被插入到索引中。这些记录使用了与主表不同的键，这个键在使用`WT_SESSION::create`方法创建索引时被指定。

在索引上打开的游标使用指定的索引列作为它的键，通过`WT_CURSOR::set_key`和`WT_CURSOR::get_key`访问。值的列默认返回表中值的列，但是通过配置投影游标（参见[投影]()）可以覆盖--可以访问表的键的列或者值的列的子集。

**统计信息数据**

统计信息游标可以用于检索WiredTiger数据库的运行时统计信息以及单独数据源的统计信息。统计信息有两层：每个数据库的和每个单独的数据源的。数据库范围的统计使用使用”statistics:“检索；单独数据源的统计信息通过”statistics:&lt;data source URI>“获取。

统计信息的键是整数，来源于Statistics Keys列表。统计信息游标从`WT_CURSOR::get_value`调用中返回3个值：统计信息的可打印的描述，条目的值的可打印版本和条目的无符号64位整数值。

下面的例子打印出WiredTiger引擎的运行时统计信息：
```c
if ((ret = session->open_cursor(session, "statistics:", NULL, NULL, &cursor)) != 0) {
    return (ret);
}

ret = print_cursor(cursor);
ret = cursor->close(cursor);
```

下面的例子打印出一张表的统计信息：
```c
if ((ret = session->open_cursor(session, "statistics:table:access", NULL, NULL, &cursor)) != 0) {
    return (ret);
}

ret = print_cursor(cursor);
ret = cursor->close(cursor);
```

这两个例子都使用了一个显示子程序，这个子程序遍历统计信息直到结尾。
```c
int
print_cursor(WT_CURSOR *cursor)
{
    const char *desc, *pvalue;
    uint64_t value;
    int ret;

    while ((ret = cursor->next(cursor)) == 0 &&
           (ret = cursor->get_value(cursor, &desc, &pvalue, &value)) == 0) {
        if (value != 0) {
            printf("%s=%s\n", desc, pvalue);
        }
    }

    return (ret == WT_NOTFOUND ? 0 : ret);
}
```

单独的统计信息值可以通过搜索对应的键来检索，如下所示：
```c
WT_CURSOR *cursor;
const char *desc, *pvalue;
uint64_t value;
int ret;

if ((ret = session->open_cursor(session, "statistics:table:access", NULL, NULL, &cursor)) != 0) {
    return (ret);
}

cursor->set_key(cursor, WT_STAT_DSCR_BTREE_OVERFLOW);
ret = cursor->search(cursor);
ret = cursor->get_value(cursor, &desc, &pvalue, &value);
printf("%s=%s\n", desc, pvalue);

ret = cursor->close(cursor);
```
