命令行工具
==========

WiredTiger包含一个命令行工具wt。

**简介**  
wt [-rVv] [-C config] [-h directory] command [command-specific arguments]

**描述**  
wt是一个提供了使用WiredTiger各种功能的命令行工具。

**选项**  
有四个全局选项：  
-C config  
&nbsp;&nbsp;&nbsp;&nbsp;指定`wiredtiger_open`函数的配置字符串。  
-h directory  
&nbsp;&nbsp;&nbsp;&nbsp;指定数据库home路径。  
-r  
&nbsp;&nbsp;&nbsp;&nbsp;如果底层数据库配置了recovery，那么就执行。  
-V  
&nbsp;&nbsp;&nbsp;&nbsp;显示WiredTiger版本号并退出。  
-v  
&nbsp;&nbsp;&nbsp;&nbsp;设置详细输出。  

如果没有特别指出，wt退出时返回0表示执行成功，非0表示出错。

wt工具指出多种命令。如果在底层数据库做了配置，部分命令会在打开数据库时执行recovery。如果用户想要在任何命令上都强制执行recovery，那么使用-r选项。一般情况下，修改数据库或表的命令会默认执行recovery，而只读取数据的命令不会执行recovery。

------
**wt backup**  

在数据库或者数据源集合上执行备份。  
backup命令用于备份数据库，复制数据库文件到指定的路径，以后可以作为WiredTiger数据库打开。  

**简介**  
wt [-rVv] [-C config] [-h directory] backup [-t uri] directory  

**选项**  
-t uri
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下，backup命令备份整个数据库；-t选项使backup命令改为只在指定的数据源上做备份。

------
**wt compact**  

压缩表或文件。  

compact命令尝试重写指定的表或文件以减少占用的磁盘空间。  

**简介**  
wt [-rVv] [-C config] [-h directory] compact uri  

**选项**  
无。

------
**wt create**  

创建表或文件。  

create命令使用指定的配置创建指定的uri。它等价于使用指定的字符串参数调用`WT_SESSION::create`。  

**简介**  
wt [-rVv] [-C config] [-h directory] create [-c config] uri  

**选项**  
-c  
&nbsp;&nbsp;&nbsp;&nbsp;包含一个传给`WT_SESSION::create`的配置字符串。

------
**wt drop**  

删除表或文件。  

drop命令删除指定的uri。它等价于使用“force”配置参数调用`WT_SESSION::drop`。  

**简介**  
wt [-rVv] [-C config] [-h directory] drop uri  

**选项**  
无。

------
**wt dump**  

以文本格式导出数据。  

dump命令将指定表的数据以可移植的格式导出，导出的数据可以使用load命令重新加载到新的表中。

**简介**  
wt [-rVv] [-C config] [-h directory] dump [-jrx] [-c checkpoint] [-f output] uri  

**选项**  
-c  
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下dump命令打开数据源的最新版本；-c选项可以改变dump命令，打开命名的checkpoint。
-f  
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下dump命令输出到标准输出上；-f选项将输出重定向到指定的文件。  
-j  
&nbsp;&nbsp;&nbsp;&nbsp;以JSON格式导出。  
-r  
&nbsp;&nbsp;&nbsp;&nbsp;逆序导出，重最大的键到最小的。  
-x  
&nbsp;&nbsp;&nbsp;&nbsp;以16进制编码导出所有的字符（默认是未编码的可打印字符）。

------
**wt list**  

列出数据库中的表和文件。

默认情况下，list命令打印出存储在数据库中的所有的表和文件。如果在参数中指定了URI，则只打印指定数据源的信  息。  

**简介**  
wt [-rVv] [-C config] [-h directory] list [-cv] [uri]  

**选项**  
-c  
&nbsp;&nbsp;&nbsp;&nbsp;如果指定了-c，则将数据源的checkpoint以可读的格式打印出来。  
-v  
&nbsp;&nbsp;&nbsp;&nbsp;如果指定了-v，则将数据源的完整的模式表的值打印出来。  

------
**wt load**  

将导出数据导入到表或文件中。  

load命令从标准输入读取数据并将数据导入到表或文件中，如果表或文件不存在则创建。数据应该是由dump命令生成的。默认情况下，如果表或文件已经存在，表和文件中的数据会被新数据覆盖（使用-n选项在试图覆盖数据时返回错误）。

**简介**  
wt [-rVv] [-C config] [-h directory] load [-ajn] [-f input] [-r name] [uri configuration...]

**选项**  
-a  
&nbsp;&nbsp;&nbsp;&nbsp;如果-a被指定，输入中的记录数值键会被忽略，数据会被赋于一个新的记录数值键并添加到数据源中。该选择只适用于加载数据到列存时。  
-f
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下load命令从标准输入读取数据；-f选项从指定文件中读取输入数据。  
-j  
&nbsp;&nbsp;&nbsp;&nbsp;导入由dump -j命令生成的JSON格式的数据。  
-n  
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下当数据源中已存在键值对时输入数据会覆盖已经存在的数据；-n选项会使试图覆盖已存在数据的load命令失败。  
-r  
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下load命令使用输入数据中的表名或文件名；-r选项重命名数据源。  

另外，uri和配置参数对也可以指定给load命令。所有的配置参数对都会添加到传递给`WT_SESSION::create`的配置参数字符串中。

------
**wt loadtext**

将文本导入到表或文件中。  

loadtext命令从标准输入中读取文本并导入到表或文件中。输入数据时可打印的字符，通过换行符分割每个键值对。

loadtext命令不会创建不存在的文件。

在向列存表或文件插入值时，每个值都被添加到表或文件中；在向行存表或文件插入值时，行按对处理，第一行是键而第二行时值。如果行存表或文件已经存在，表或文件中已经存在的数据会被新数据覆盖。

**简介**  
wt [-rVv] [-C config] [-h directory] loadtext [-f input]

**选项**  
-f  
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下loadtext命令从标准输入读取数据；-f选项从指定的文件中读取数据。

------
**wt printlog**  

显示数据库日志。

**简介**  
wt [-rVv] [-C config] [-h directory] printlog [-p] [-f output]

**选项**  
-f  
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下printlog命令输出到标准输出；-f选项将输出重定向到指定文件。  
-p  
&nbsp;&nbsp;&nbsp;&nbsp;以可打印的格式显示日志。

------
**wt read**

从表或文件中读取记录。

read命令从指定数据源中读取指定的key的记录并打印出来。数据源必须通过字符串或记录的数值键和字符串值来配置。

如果未找到指定记录，read命令以非0值退出。

**简介**  
wt [-rVv] [-C config] [-h directory] read uri key ...

**选项**  
无。

------
**wt rename**  
重命名表或文件。

**简介**
wt [-rVv] [-C config] [-h directory] rename uri name

**选项**  
无。

------
**wt salvage**

从损坏的文件中恢复数据。

salvage命令抢救指定的数据源，丢弃不能恢复的数据。底层文件被重写，覆盖原始的文件内容。

**简介**  
wt [-rVv] [-C config] [-h directory] salvage [-F force] uri

**选项**  
-F  
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下salvage命令会拒绝抢救未通过基本测试的文件（例如：文件看起来不是WiredTiger格式）。-F选项强制恢复文件。

------
**wt stat**

显示数据库或数据源的统计信息。

stat命令输出WiredTiger引擎的运行时统计信息，或者通过命令行指定的URI的统计信息。

**简介**  
wt [-rVv] [-C config] [-h directory] stat [-f] [uri]

**选项**  
-f  
&nbsp;&nbsp;&nbsp;&nbsp;只输出"fast"统计信息（等价于传递`statistics=(fast)`给`WT_SESSION::open_cursor`）。

------
**wt upgrade**

升级表或文件，如果数据源是最新的则成功，如果数据源不能被升级则失败。

**简介**  
wt [-rVv] [-C config] [-h directory] upgrade uri

**选项**  
无。

------
**wt verify**

检查表或文件的结构完整性，如果数据源是正确的则成功，如果数据源损坏则失败。

**简介**  
wt [-rVv] [-C config] [-h directory] verify uri

**选项**  
无。

------
**wt write**

向表或文件中写入记录。

试图覆盖已存在的记录会导致失败。

**简介**  
wt [-rVv] [-C config] [-h directory] write -a uri value ...  
wt [-rVv] [-C config] [-h directory] write [-o] uri key value ...

**选项**  
-a  
&nbsp;&nbsp;&nbsp;&nbsp;把每个值作为一个新记录添加到数据源中。  
-o  
&nbsp;&nbsp;&nbsp;&nbsp;默认情况下试图覆盖已存在的记录会导致失败。-o选项会覆盖已存在的记录
