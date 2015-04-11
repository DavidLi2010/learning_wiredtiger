数据库HOME路径
=============

WiredTiger数据库HOME路径由`wiredtiger_open`函数的home参数和环境变量WIREDTIGER_HOME决定，根据以下步骤：<br>
1. 如果指定了`wiredtiger_open`函数的home参数，那么使用home作为数据库HOME路径；<br>
2. 如果没有设置环境变量WIREDTIGER_HOME，那么数据库HOME路径是进程的当前工作路径。WiredTiger不维护当前工作路径，在打开WiredTiger数据库后改变工作路径会导致错误；<br>
3. 如果进程以特殊权限运行，那么`wiredtiger_open`函数必须配置`use_environment_priv`标识。`use_environment_priv`标识用于拥有或者获取了特殊权限并且希望使用环境相关的home路径的应用程序。如果这样的应用程序没有为`wiredtiger_open`函数配置`use_environment_priv`标识，那么会导致打开失败；<br>
4. 否则，会使用环境变量WIREDTIGER_HOME的值作为数据库路径。<br>

使用环境变量WIREDTIGER_HOME时要考虑安全问题，特别是使用的不是user权限的应用程序。
