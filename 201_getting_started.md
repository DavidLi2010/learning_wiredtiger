开始使用API
===========
WiredTiger的应用程序通常使用下面的类来访问和管理数据：
- WT_CONNECTION表示到数据库的连接。大多数应用在每个进程上对一个数据库只会打开一个连接。WT_CONNECTION的所有方法都是线程安全的。
- WT_SESSION表示执行数据库操作的上下文。会话在一个连接上被打开，应用程序访问数据库时必须为每一个线程打开一个会话。
- WT_CURSOR表示在数据集合上的游标。游标在会话的上下文中被打开（可能有相关的事务），可以查询和更新记录。通常，游标用于访问一张表上的记录。然而
然而，游标也可以用在表的子集上（例如多个列上的一个列或投影），作为对静态配置数据或应用相关数据源的接口。

句柄和操作通过字符串配置，使得API中的方法集保持相对较小，并且无论应用程序使用何种编程语言都有类似的接口。WiredTiger支持C、C++、Java和Python编程语言。

默认情况下，WiredTiger被用于传统的key/value存储，其中键和值都是原始的字节数组并通过WT_ITEM结构来访问。键和值可以达到(4GB-512B)字节大小，但依赖WT_SESSION::create的"maximum item size"的配置，大的键和值会被保存到溢出页上。

WiredTiger支持一个schema层，因此键和值的类型可以从列表中选择，或者由任何类型组合而成的列组成的复合键和值。对键和值的字节大小的限制仍然有效。

所有使用WiredTiger的应用程序大概按如下方式组织。 下面的代码是从示例程序ex_access.c中摘录的。

**连接到数据库**

要访问数据库，首先打开一个连接并为访问数据库的线程创建一个会话句柄：
```c
WT_CONNECTION *conn;
WT_CURSOR *cursor;
WT_SESSION *session;
const char *key, *value;
int ret;
/*
* Create a clean test directory for this run of the test program if the
* environment variable isn't already set (as is done by make check).
*/
if (getenv("WIREDTIGER_HOME") == NULL) {
       home = "WT_HOME";
       ret = system("rm -rf WT_HOME && mkdir WT_HOME");
} else
       home = NULL;
/* Open a connection to the database, creating it if necessary. */
if ((ret = wiredtiger_open(home, NULL, "create", &conn)) != 0 ||
   (ret = conn->open_session(conn, NULL, NULL, &session)) != 0) {
       fprintf(stderr, "Error connecting to %s: %s\n",
           home, wiredtiger_strerror(ret));
       return (ret);
}
```
配置字符串"create"传给`wiredtiger_open`表示如果数据库不存在则创建它。

上面的代码块同时展示了使用`wiredtiger_strerror`（错误码作为参数，返回字符串描述）进行简单的错误处理。更复杂的错误处理可以通过传递一个WT_EVENT_HANDLER的实现给`wiredtiger_open`或`WT_CONNECTION::open_session`。

**创建表**

创建表之后可以存储数据：
```c
ret = session->create(session,
            "table:access", "key_format=S,value_format=S");
```

这次调用创建了名为"access"的表，配置它的键和值使用字符串。
