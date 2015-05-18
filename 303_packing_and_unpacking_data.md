打包与解包数据
==============
WiredTiger的数据打包使用类似在Python struct模块列举的格式字符串：<http://docs.python.org/library/struct>。

格式字符串的第一个字符可以用来说明字节序，大小和打包数据的对齐，参考下表：

|字符|字节序|大小|对齐|
|----|------|----|---|
|.|大端|打包|无|
|>|大端|标准|无|
|<|小端|标准|无|
|@|本地|本地|本地|

如果第一个字符不是以上中的一个，那么就假定为'.'（大端，打包）：按字典序排序，打包格式使用变长编码的值以减少数据大小。

注意：WiredTiger目前不支持**小端格式**。目前只支持默认的大端，打包格式。

格式字符串余下的字符说明打包到字节数组或者从字节数组解包的每个字段的类型。查看[列类型]()以获取支持的类型。

**代码示例**

下面的代码从ex_pack.c中摘取。它说明了如何将3个整数值打包到缓存中，然后又从中解包。
```c
size_t size;
char buf[50];

ret = wiredtiger_struct_size(session, &size, "iii", 42, 1000, -9);
if (size > sizeof(buf)) {
    /* Allocate a bigger buffer. */
}

ret = wiredtiger_struct_pack(session, buf, size, "iii", 42, 1000, -9);

ret = wiredtiger_struct_unpack(session, buf, size, "iii", &i, &j, &k);
```
