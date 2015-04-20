配置字符串
==========
**简介**

WiredTiger的方法通过使用配置字符串来提供可选参数和配置非标准的行为。这些字符串是用逗号隔开的简单的"&lt;key>=&lt;value>"对，都有相同的格式：  
`[key['='value]][','[key['='value]]]*`

键和值以及由字母数字字符组成的值可以直接指定。更准确地说，匹配下面正则表达式的键或值不需要引号：  
`[-_0-9A-Za-z./][^\t\r\n :=,\])}]*`  

更复杂的键和值可以用双引号引用。

例如，当打开到数据库的连接时，使用配置字符串指定数据库是否应该被创建以及设置缓存大小：
```c
ret = wiredtiger_open(home, NULL, "create,cache_size=500M", &conn);
```
配置字符串也可以配置表的模式。例如，配置一个键和值使用C语言字符串的表：
```c
ret = session->create(session, "table:mytable", "key_format=S,value_format=S");
```
为了处理更复杂的模式配置，例如指定表的多个列，使用括号嵌套值。例如：
```c
/*
 * Create a table with columns: keys are record numbers, values are
 * (string, signed 32-bit integer, unsigned 16-bit integer).
 */
ret = session->create(session, "table:mytable",
      "key_format=r,value_format=SiH,columns=(id,department,salary,year-started)");
```
解析器平等地对待括号内的类型。

当希望有一个整数类型的值时，值可以有附加的乘数字符，如下：

|字符|意义|改变数值|
|----|----|---------------|
|B或b|byte|无变化|
|K或k|kilobyte|乘以2^10|
|M或m|megabyte|乘以2^20|
|G或g|gigabyte|乘以2^30|
|T或t|terabyte|乘以2^40|
|P或p|petabyte|乘以2^50|

例如,值500B与输入数值500是相同的，值500K等同于512000，500GB等同于536870912000。

布尔型的值可以设为false，true，0或1。如果没有指定key的值，默认为1。例如，下面所有的overwrite配置字符串是相同的：
```c
ret = session->open_cursor(session, "table:mytable", NULL,
      "overwrite", &cursor);
ret = session->open_cursor(session, "table:mytable", NULL,
      "overwrite=true", &cursor);
ret = session->open_cursor(session, "table:mytable", NULL,
      "overwrite=1", &cursor);
```

配置字符串被从左到右按顺序处理，后面的设置会覆盖前面的（除非该方法支持对同一个key的多次设置）。

配置字符串中多余的逗号和空白会被忽略（包括字符串的开头和结尾），所以使用逗号将两个配置字符串连接起来总是安全的。

在C或C++中可以传递NULL表示空的配置字符串。

**JSON兼容性**  

WiredTiger的配置字符串与JSON兼容，能够接受下面的格式：
- 括号可以是圆括号或者方括号或者花括号：'()', '[]', '{}'
- 整个配置字符串可以包在括号中
- 键和值之间可以用':'分隔
- 键和值可以用双引号引起来："key" = "value"
- 引起来的字符串按UTF-8解析

这种宽松的解析的结果是无论哪里需要配置字符串，应用程序都可以传递代表有效的JSON对象的字符串。
例如，在Python中，代码类似于：
```python
import json
config = json.dumps({
    "key_format" : "r",
    "value_format" : "5sHQ",
    "columns" : ("id", "country", "year", "population"),
    "index.country_year" : ["country", "year"]
})
```

因为JSON的兼容性允许使用冒号做键和值的分隔符，WiredTiger的URI可能需要引起来。例如，下面调用`WT_SESSION::checkpoint`时指定了一组URI作为checkpoint的目标，使用双引号以使解析器不把冒号作为JSON键和值的分隔符：
```c
/*
 * Checkpoint a list of objects.
 * JSON parsing requires quoting the list of target URIs.
 */
 ret = session->checkpoint(session, "target=(\"table:table1\", \"table:table2\")");
```
