游标
====
WiredTiger中的常见操作是通过WT_CURSOR句柄来执行的。游标包括：
- 在数据源中的位置
- 键和值的getter/setter方法
- 将字段编码以存储到数据源中
- 在数据源中操纵与遍历的方法

在[游标操作]()中查看对如何使用游标的描述。

**游标类型**

下面是内置的基本游标类型：

|URI|类型|说明|
|---|----|----|
|table:&lt;table name>[&lt;projection>]|表游标|table key, table value(s) with optional projection of columns|
|colgroup:&lt;table name>:&lt;colume group name>|列组游标|table key, column group value(s)|
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

**投影**

在表和索引上的游标可以返回列的子集。通过在给`WT_SESSION::open_cursor`的uri参数中给出用括号括起的列名可以实现。`WT_CURSOR::get_value`只返回给定列的字段。

ex_schema.c示例创建了一个表，值的格式是“5sHq”，开头的字符串是国家，short类型的是年份，long类型的是人口。下面的例子从表的记录值中只列出国家和年份这两列：
```c
/*
 * Use a projection to return just the table's country and year
 * columns.
 */
ret = session->open_cursor(session, "table:poptable(country, year)", NULL, NULL, &cursor);
while ((ret = cursor->next(cursor)) == 0) {
    ret = cursor->get_value(cursor, &country, &year);
    printf("country %s, year %u\n", country, year);
}
```
这在使用索引游标时特别有用，因为如果投影中所有的列都能在索引中得到（包括主键列，也是索引的值），数据就可以直接从索引中读取而不用访问列组。

**游标和事务**

如果一个会话中有活跃的事务，那么游标就在事务的上下文中操作。当事务是活跃的时候，读取操作继承了事务的隔离级别，而事务中的更新操作通过调用`WT_SESSION::commit_transaction`使得持久化，或者通过调用`WT_SESSION::rollback_transaction`而丢弃。

如果没有活跃的事务，游标的读取操作使用会话的隔离级别，为`WT_CONNECTION::open_session`设置`isolation`配置项，执行成功的更新会在更新操作完成前自动提交。

由多个相关的更新组成的操作都应该被放入显式的事务中，以确保这些更新的原子性。

在read-committed（默认）或snapshot隔离级别，当没有游标已确定位置时在并发的事务中已提交的更改是可见的。换句话说，在这两个隔离级别，在同一个会话中的所有的游标从一个稳定的快照中读数据，而同时任何游标都保持自己的位置。

游标的位置比事务的边界存在时间要长，除非事务回滚了。当事务被隐式或显式地回滚，会话上的所有游标都被复位，就像调用了`WT_CURSOR::reset`一样，丢弃了所有游标的位置和所有的键和值。

**Raw模式**

通过设置”raw“配置关键字到`WT_SESSION::open_cursor`，可以将游标配置为raw模式。在这个模式，方法`WT_CURSOR::get_key`，`WT_CURSOR::get_value`，`WT_CURSOR::set_key`和`WT_CURSOR::set_value`都从变长参数列表中获取一个`WT_ITEM`，代替按每一列分开的参数。

`WT_ITEM`结构在使用前不需要清理。

对于在raw模式的`WT_CURSOR::get_key`和`WT_CURSOR::get_value`，通过调用`WT_EXTENSION_API::struct_unpack`，使用游标的key_format或value_format，`WT_ITEM`可以被分裂成列。对于在raw模式的`WT_CURSOR::set_key`和`WT_CURSOR::set_value`，`WT_ITEM`应该与使用游标的key_format或value_format调用`WT_EXTENSION_API::struct_pack`相同。

ex_schema.c示例创建了一个表，它的值格式是”5sHq“，开头的字符串是国家，short类型的是年份，long类型的是人口。下面的例子使用raw模式列出表中记录的值：
```c
/* List the records in the table using raw mode. */
ret = session->open_cursor(session, "table:poptable", NULL, "raw", &cursor);
while ((ret = cursor->next(cursor)) == 0) {
    WT_ITEM key, value;
	
	ret = cursor->get_key(cursor, &key);
	ret = wiredtiger_struct_unpack(session, key.data, key.size, "r", &recno);
	printf("ID %" PRIu64, recno);
	
	ret = cursor->get_value(cursor, &value);
	ret = wiredtiger_struct_unpack(session, value.data, value.size, "5sHq", &country, &year, &population);
	printf(": country %s, year %u, population %" PRIu64 "\n", country, year, population);
}
```

raw模式可以与投影组合使用。下面的例子使用raw模式只列出记录值中年份和人口两列：
```c
/*
 * Use a projection to return just the table's country and year
 * columns, using raw mode.
 */
ret = session->open_cursor(session, "table:poptable(country, year)", NULL, "raw", &cursor);
while ((ret = cursor->next(cursor)) == 0) {
    WT_ITEM value;
	
	ret = cursor->get_value(cursor, &value);
	ret = wiredtiger_struct_unpack(session, value.data, value.size, "5sH", &country, &year);
	printf("country %s, year %u\n", country, year);
}
```

**读取WiredTiger的元数据**

WiredTiger的游标提供了访问各类数据源的能力。其中一类数据源就是数据库中对象的列表。

为了检索数据库对象的列表，在`metadata:`这个URI上打开游标。返回的每一个键是数据库对象，每个值是对象的元数据信息。

例如：
```c
ret = session->open_cursor(session, "metadata:", NULL, NULL, &cursor);
```

元数据游标是只读的，元数据不能被修改。

