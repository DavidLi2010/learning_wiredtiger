数据库配置
=========

WiredTiger在调用`wiredtiger_open`函数的时候通过配置字符串参数来配置数据库。
另外，也可以使用指定的WiredTiger.config文件或者WIREDTIGER_CONFIG环境变量。

当创建WiredTiger数据库时，通过`wiredtiger_open`传递的配置字符串被保存到WiredTiger的home路径下，文件名为WiredTiger.basecfg。

调用`wiredtiger_open`时传入的配置字符串使得应用程序在每次运行时设置或者覆盖创建时的原始设置。 用户的配置文件和环境变量允许系统管理员覆盖应用设置而不需重新编译。

###配置顺序
当数据库被创建或打开时，配置顺序：
- 任何WiredTiger.basecfg文件
- `wiredtiger_open`的配置字符串参数覆盖上面的配置
- WiredTiger.config文件覆盖上面的配置
- WIREDTIGER_CONFIG环境变量覆盖上面的配置

###WIREDTIGER_CONFIG环境变量
如果设置了WIREDTIGER_CONFIG环境变量，按字符串读取它的值。

###WiredTiger.config文件
如果在WiredTiger的home路径中有名为WiredTiger.config的文件，读取它的内容作为配置字符串。<br>
这个文件被最小限度的解析来为WiredTiger配置解析器生成配置字符串：
- 反斜线(\\)之后的任何字符（除了换行符）都不会改变；否则，如果反斜线后跟着换行符，反斜线和换行符都会被丢弃。
- 双引号(")对内的任何文本都不会改变，包括换行符和空白符。反斜线转义双引号：反斜线转义的双引号符既不能表示字符串的开始也不能表示结束。
- 注释被丢弃。如果第一个非空白字符是#，那么直到下一个换行符的所有字符都被丢弃。最后的换行符不能被转义或括起。
- 否则，所有的行被连接起来，换行符被替换为逗号。

###WiredTiger.basecfg文件
当数据库被创建时，所有的非默认的配置信息都会保存到WiredTiger的home路径下名为WiredTiger.basecfg的文件中。以后无论何时打开数据库都会读取该文件。
