模式、列、列组、索引和投影
=========================
WiredTiger的许多表有简单的键值对记录，而同时也支持更复杂的数据模式。

**表、行和列**

表是对由行和列中的单元组成的数据的逻辑表述。例如，一个数据库可能有一个简单的表，包括雇员ID、名字、姓、薪水：

|雇员ID|名字|姓|薪水|
|------|----|--|----|
|1|Smith|Joe|40000|
|2|Jones|Mary|50000|
|3|Johnson|Cathy|44000|

行式数据库会按行存储所有的数据，第一个雇员在第一行，然后下一个雇员的数据在下一行，等等：

    1,Smith,Joe,40000
    2,Jones,Mary,50000
    3,Johnson,Cathy,44000

列式数据库会把所有数据的列存储到一起，然后是数据的下一个列，等等：

    1,2,3
    Smith,Jones,Johnson
    Joe,Mary,Cathy
    40000,50000,44000

WiredTiger支持这两种存储格式，并且可以在一个逻辑表中混合和匹配列存。

WiredTiger的表由一个或多个列组组成，按主键的顺序保存所有的列，并有零个或多个索引加速按列的顺序而不是按主键查询。

应用程序通过为`WT_SESSION::create`提供一个模式来描述数据的格式，说明了应用程序的数据如何分裂成字段以及映射到行和列。

**列类型**

默认情况下，WiredTiger像传统的键/值存储那样工作，键和值是原始的字节数组，可以使用WT_ITEM访问。键和值的类型可以从一个列表中选择，或者由多个使用任何类型组合而成的列组成。键和值的大小可以是(4GB-512B)。

关于原始的键/值项的更多详细信息可以查看[键值对]()。

**格式类型**

WiredTiger使用的格式字符串与Python的struct模块描述的类型类似：<http://docs.python.org/library/struct>

|格式|C类型|Java类型|Python类型|说明|
|----|-----|--------|----------|---|
|x|N/A|N/A|N/A|对齐字节，没有值|
|b|int8_t|byte|int|有符号字节|
|B|uint8_t|byte|int|无符号字节|
|h|int16_t|short|int|有符号16位|
|H|uint16_t|short|int|无符号16位|
|i|int32_t|int|int|有符号32位|
|I|uint32_t|int|int|无符号32位|
|l|int32_t|int|int|有符号32位|
|L|uint32_t|int|int|无符号32位|
|q|int64_t|long|int|有符号64位|
|Q|uint64_t|long|int|无符号64位|
|r|uint64_t|long|int|记录编号|
|s|char[]|String|str|定长字符串|
|S|char[]|String|str|以NUL结束的字符串|
|t|uint8_t|byte|int|定长位域|
|u|WT_ITEM*|byte[]|str|原始字节数组|

**键和值的格式**

**游标格式**

**列**

**列组**

**索引**

**不可变索引**

**索引游标投影**

**示例代码**
