游标操作
========
WiredTiger中的常见操作是通过WT_CURSOR句柄来执行的。游标包括：
- 在数据源中的位置
- 键和值的getter/setter方法
- 将字段编码以存储到数据源中
- 在数据源中操纵与遍历的方法

**打开游标**

游标是由方法`WT_SESSION::open_cursor`创建的。例如，在程序ex_cursor.c中:
```c
ret = session->open_cursor(session, "table:world", NULL, NULL, &cursor);
```
在此程序中还有其它的例子：
```c
ret = session->open_cursor(session, "table:world(country, population)", NULL, NULL, &cursor);
```
另外对于传统的数据源，WiredTiger的游标用于访问投影，甚至创建像运行时统计信息这样的数据源：
```c
ret = session->open_cursor(session, "statistics:", NULL, NULL, &cursor);
```

查看[游标](203_cursors.md)获取关于游标类型的更多信息。

**关闭游标**

游标一直保持打开直到调用`WT_CURSOR::close`或者游标的会话被关闭，可通过`WT_SESSION::close`或者`WT_CONNECTION::close`关闭会话。

**游标定位**

游标可以定位到数据源的开头，数据源的结尾，数据源内的正确的键，数据源内键的周围。

为了使随后的遍历从数据源的开头或结尾开始，使用`WT_CURSOR::reset`方法将游标的位置设为无效:
```c
int
cursor_reset(WT_CURSOR *cursor)
{
    return (cursor->reset(cursor));
}
```

要在数据源中向前移动游标，使用`WT_CURSOR::next`方法：
```c
int
cursor_forward_scan(WT_CURSOR *cursor)
{
    const char *key, *value;
    int ret;

    while ((ret = cursor->next(cursor)) == 0) {
        ret = cursor->get_key(cursor, &key);
        ret = cursor->get_value(cursor, &value);
    }
    return (ret);
}
```
如果游标没有定位在数据源上，在游标上调用`WT_CURSOR::next`方法会将游标定位到数据源的开始位置。

要在数据源中反向移动游标，使用`WT_CURSOR::next`方法：
```c
int
cursor_reverse_scan(WT_CURSOR *cursor)
{
    const char *key, *value;
    int ret;

    while ((ret = cursor->prev(cursor)) == 0) {
        ret = cursor->get_key(cursor, &key);
        ret = cursor->get_value(cursor, &value);
    }
    return (ret);
}
```
如果游标没有定位在数据源上，在游标上调用`WT_CURSOR::prev`方法会将游标定位到数据源的结尾位置。

要将游标定位到数据源中的特定位置，使用`WT_CURSOR::search`方法：
```c
int
cursor_search(WT_CURSOR *cursor)
{
    const char *value;
    int ret;

    cursor->set_key(cursor, "foo");

    if ((ret = cursor->search(cursor)) != 0) {
        ret = cursor->get_value(cursor, &value);
    }

    return (ret);
}
```

要将游标定位到数据源中的某个位置或者其周围，使用`WT_CURSOR::search_near`方法：
```c
int
cursor_search_near(WT_CURSOR *cursor)
{
    const char *key, *value;
    int exact, ret;

    cursor->set_key(cursor, "foo");

    if ((ret = cursor->search_near(cursor, &exact)) == 0) {
        switch(exact) {
        case -1:
            /* Returned key smaller than search key */
            ret = cursor->get_key(cursor, &key);
            break;
        case 0:
            /* Exact match found */
            break;
        case 1:
            /* Returned key larger than search key */
            ret = cursor->get_key(cursor, &key);
            break;
        }

        ret = cursor->get_value(cursor, &value);
    }

    return (ret);
}
```

游标的位置不会比事务存在的时间更长：在`WT_SESSION::begin_transaction`期间打开的游标，`WT_SESSION::commit_transaction`或`WT_SESSION::rollback_transcation`会使得它们丢失位置，就像调用了`WT_CURSOR::reset`一样。

游标可以配置成调用`WT_CURSOR::next`时移动到随机的位置，参见[游标随机]()。

**插入和更新**

为了插入新数据，以及有选择的更新已存在的数据，使用`WT_CURSOR::insert`方法：
```c
int
cursor_insert(WT_CURSOR *cursor)
{
    cursor->set_key(cursor, "foo");
    cursor->set_value(cursor, "bar");

    return (cursor->insert(cursor));
}
```

为了更新已存在的数据，使用`WT_CURSOR::update`方法：
```c
int
cursor_update(WT_CURSOR *cursor)
{
    cursor->set_key(cursor, "foo");
    cursor->set_value(cusor, "bar");

    return (cursor->update(cursor));
}
```

为了删除已存在的数据，使用`WT_CURSOR::remove`方法：
```c
int
cursor_remove(WT_CURSOR *cursor)
{
    cursor->set_key(cursor, "foo");
    return (cursor->remove(cursor));
}
```

`WT_SESSION::open_cursor`的overwrite配置项默认为true，使得`WT_CURSOR::insert`，`WT_CURSOR::update`和`WT_CURSOR::remove`会忽略记录的当前状态，这些方法会执行成功而不管记录是否存在。

当应用程序当overwrite配置成false，如果记录已存在，`WT_CURSOR::insert`会产生`WT_DUPLICATE_KEY`错误，如果记录不存在，`WT_CURSOR::update`和`WT_CURSOR::remove`会产生`WT_NOTFOUND`错误。

**出错后的游标位置**

在任何游标的方法失败后，游标的位置是未知的。对于那些在操作开始前需要设置键的游标操作（包括`WT_CURSOR::search`，`WT_CURSOR::insert`，`WT_CURSOR::update`和`WT_CURSOR::remove`），出错是应用程序的键和值不会被清除。

如果应用程序在出错后不能重定位游标，那么必须在调用那些会重定位游标的方法前调用`WT_SESSION::open_cursor`复制游标，并把游标作为to_dup参数。备份、配置和统计信息类型的游标不支持游标复制。

**游标的键/值的内存范围**

当应用程序传递指针（可能是WT_ITEM或者字符串）给`WT_CURSOR::set_key`或者`WT_CURSOR::set_value`时，应用程序要保证内存有效，直到下一个操作成功定位游标。这些操作是`WT_CURSOR::remove`，`WT_CURSOR::search`，`WT_CURSOR::search_near`和`WT_CURSOR::update`，但是不包括`WT_CURSOR::insert`，因为它不会定位游标。

如果这些操作失败（例如，WT_ROLLBACK错误），可能会重试而不会再次调用`WT_CURSOR::set_key`或`WT_CURSOR::set_value`。即是说，游标可能会一直引用应用程序提供的内存直到成功定位。

任何由`WT_CURSOR::get_key`或`WT_CURSOR::get_value`返回的指针只在游标定位后有效。这些指针可能引用了WiredTiger私有的数据结构，不能被应用程序修改或释放。如果需要更久的范围，应用程序必须在游标定位前复制内存。

下面例子中的注释解释了应用程序什么时候可以安全的修改传给`WT_CURSOR::set_key`或`WT_CURSOR::set_value`的内存：
```c
(void)snprintf(keybuf, sizeof(keybuf), "%s", op->key);
cursor->set_key(cursor, keybuf);
(void)snprintf(valuebuf, sizeof(valuebuf), "%s", op->value);
cursor->set_value(cursor, valuebuf);

/*
* The application must keep the key and value memory valid
* until the next operation that positions the cursor.
* Modifying either the key or value buffers is not permitted.
*/

/* Apply the operation (insert, update, search or remove). */
if ((ret = op->apply(cursor)) != 0) {
    fprintf(stderr, "Error performing the operation: %s\n", wiredtiger_strerror(ret));
    return (ret);
}

/*
* Except for WT_CURSOR::insert, the cursor has been positioned
* and no longer references application memory, so application
* buffers can be safely overwritten.
*/
if (op->apply != cursor->insert) {
    strcpy(keybuf, "no key");
    strcpy(valuebuf, "no value");
}

/*
* Check that get_key/value behave as expected after the
* operation.
*/
if ((ret = cursor->get_key(cursor, &key)) != 0 ||
    (op->apply != cursor->remove && (ret = cursor->get_value(cursor, &value)) != 0)) {
    fprintf(stderr, "Error in get_key/value: %s\n", wiredtiger_strerror(ret));
    return (ret);
}

/*
* Except for WT_CURSOR::insert (which does not position the
* cursor), the application now has pointers to memory owned
* by the cursor. Modifying the memory referenced by either
* key or value is not permitted.
*/

/* Check that the cursor's key and value are what we expect. */
if (op->apply != cursor->insert) {
    if (key == keybuf ||
        (op->apply != cursor->remove && value == valuebuf)) {
        fprintf(stderr, "Cursor points at application memory!\n");
        return (EINVAL);
    }
}

if (strcmp(key, op->key) != 0 ||
    (op->apply != cursor->remove && strcmp(value, op->value) != 0)) {
    fprintf(stderr, "Unexpected key/value!\n");
    return (EINVAL);
}
```
