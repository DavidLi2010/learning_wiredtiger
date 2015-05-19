日志结构合并树（LSM树）
=====================
**背景**

一个常见的需求是在随机插入的工作负载下保持吞吐率，键的范围经过选择因此不太可能重复（例如，128位的hash），或者插入可以覆盖已经存在的值。

对于传统的B树的变种，当数据集保持在缓存中时，插入非常快。但是一旦树溢出了缓存，性能下降很严重。这主要有两个因素：
1. 一旦数据填充到缓存，新的插入就有可能到不在缓存的页面上，这就需要读磁盘；
2. 缓存中充满了脏页，因此在读磁盘之前页面必须写到磁盘以释放缓存空间。

**LSM树的说明**

日志结构合并树在由Patrick O'Neil, Edward Cheng, Dieter Gawlick and Elizabeth O'Neil撰写的论文中说明：<http://www.cs.umb.edu/~poneil/lsmtree.pdf>。

一颗逻辑的树被分割成多个物理片段，这样树的数据中最近更新过的部分整个都在内存中。内存中数据块的大小可以由`WT_SESSION::create`的配置键“lsm=(chunk_size)”来配置。

一旦内存中的树达到了阈值大小，一个新的内存树被创建而旧的树被同步到磁盘。一旦写到磁盘，树就是只读的，即使它们在后台与磁盘上的其它树合并以减少读操作。

使用这种数据结构，可以在整个内存树上“盲写”。删除由插入一个特殊的“墓碑”记录到内存树中实现。

**LSM树的接口**

可以像下面这样创建LSM树，与WiredTiger的B树文件很像：
```c
session->create(session, "table:bucket", "type=lsm,key_format=S,value_format=S");
```

一旦创建，可以使用与WiredTiger中其它数据源相同的游标接口访问LSM树：
```c
WT_CURSOR *c;

session->open_cursor(session, "table:bucket", NULL, NULL, &c);
for(;;) {
    c->set_key(c, "key");
    c->set_value(c, "value");
    c->insert(c);
}
```

如果在调用`WT_SESSION::open_cursor`时LSM游标被配置为"overwrite=false"，在每次修改前都会搜索LSM树的所有的层。

**合并**

每次激活LSM树时一个后台线程都会被打开。这个线程负责将旧数据块写到稳定的存储上，并合并多个块以便少量文件就能满足读操作。目前没有办法配置合并：它们由后台线程自动执行。

**布隆过滤器**

当合并的时候，WiredTiger会创建一个布隆过滤器。这是一个包含每个键可配置数量的位（默认为8）的额外的文件。键被哈希多次，次数可配置（默认为4），以及对应的位集合。布隆过滤器用于在键不存在时避免从块中读取。

默认情况下，布隆过滤器只需要每个键一个字节，因此通常能够放在缓存中。布隆参数可以在`WT_SESSION::create`上使用"lsm=(bloom_bit_count)"和"lsm=(bloom_hash_count)"来配置。布隆文件可以使用"lsm=(bloom_config)"来配置。

**使用LSM树创建表**

表或者索引可以使用LSM树排序。模式支持LSM作为`WT_SESSION::create`方法的扩展：
```c
session->create(session, "table:T", "type=lsm");
```

所有模式对象的默认类型仍然是B树。

**警告**

冒险(hazard)配置

从LSM游标读取数据可能需要在每个活动的块上定位。块的数量取决于块的大小以及多少块被合并了。对于每一个在LSM树上打开的游标，至少有与树上的块相同数量的冒险指针(hazard pointer)。冒险指针的数量在调用`wiredtiger_open`时通过“hazard_max”来配置。

命名检查点

LSM树不支持命名检查点，不能使用非空的“checkpoint”配置项打开游标（即只有最近的标准检查点能够被读取）。
