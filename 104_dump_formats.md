导出格式
========
wt dump命令生成可被wt load导入的表的纯文本表示。本文说明wt dump命令的输出格式。

**JSON导出格式**

JSON导出文件使用标准的JSON数据交换格式，可以被任何JSON阅读器解析。

格式是一个JSON对象，键是传给`WT_SESSION::create`的URI，对应的值是包含两个条目的JSON数组。数组中的第一个条目是由配置信息组成的JSON对象：键“config”包含`WT_SESSION::create`使用的配置字符串，键“colgroups”和“indices”的值为配置信息组成的对象数组。第二个条目是一个JSON数组，每一个条目表示一条记录数据。如果在`WT_SESSION::create`使用的配置字符串中给出了列名，那么这些名字会被键使用，否则会生成可预见的名字（例如：“key0”，“value0”， “value1”）。对象中的值是记录中的每个列的值。

下面是示例：
```javascript
{
    "table:planets" : [
        {
            "config" : "columns=(id,name,distance),key_format=i,value_format=Si",
            "colgroups" : [],
            "indices" : [
                {
                    "uri" : "index:planets:names",
                    "config" : "columns=(name),key_format=Si,source=\"file:astronomy.wt\",type=file"
                }
            ]
        },
        [
            {
                "id" : 1,
                "name" : "Mercury",
                "distance" : 57910000
            },
            {
                "id" : 2,
                "name" : "Venus",
                "distance" : 108200000
            },
            ...
       ]
   ]
}
```

**文本导出格式**

文本导出格式有3部分，前缀、头部和正文。

导出的前缀包含基本信息，包括WiredTiger版本号和导出格式。导出格式由以“Format=”开头的行和以下信息组成：

|字符串|意义|
|-|-|
|hex|导出数据是16进制格式|
|print|导出数据时可打印格式|

导出数据的头部包含一个"Header"行和键值对组成行，键是传给`WT_SESSION::create`的URI，值是对应的配置字符串。通过为头部中的每一行键值对调用`WT_SESSION::create`，可以重建表或文件。

正文包含一个"Data"行和表中记录的文本表示。每条记录由两行表示：第一行是键，第二行是值。这些行使用两种编码格式中的一种：可打印格式和16进制格式。

可打印格式由可打印的文字字符组成，而16进制编码了不可打印的字符。编码后的字符使用3种单独的字符：反斜线后跟随两个16进制字符（第一个是高位，另一个是低位）。例如，在ASCII中回车符编码为"\0a"，而换码符编码为"\1b"。不在16进制编码前面的反斜线要成对，即"\\"被解析为单个反斜线。

16进制格式由编码后的字符组成，每个文字字符由一对字符组成（第一个是高位，另一个是低位）。例如，"0a"是ASCII的换行符，而"1b"是ASCII的换码符。

因为“可打印”的定义依赖应用的locale，以可打印格式导出的文件的可移植性可能不如以16进制格式导出的文件。
