模式、列、列组、索引和投影
=========================
WiredTiger的许多表有简单的键值对记录，而同时也支持更复杂的数据模式。

**表、行和列**

表是对由行和列中的单元组成的数据的逻辑表述。例如，一个数据库可能有一个简单的表，包括雇员ID、名字、姓、薪水：

|雇员ID|名字|姓|薪水|
|------|---|---|---|
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

关于原始的键/值项的更多详细信息可以查看[键值对](302_key_value_pairs.md)。

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
|r|uint64_t|long|int|记录号|
|s|char[]|String|str|定长字符串|
|S|char[]|String|str|以NUL结束的字符串|
|t|uint8_t|byte|int|定长位域|
|u|WT_ITEM*|byte[]|str|原始字节数组|

'r'类型在列存中用于记录号键，在其它方面与'Q'类型完全相同。

'S'类型与C语言字符串编码相同，以NUL字符结束。

't'类型用于定长的位域。在前面加上大小表示存储的位数，取值在1和8之间。低位的数值会存储在表中。默认大小是1位，即为布尔值。在调用`WT_CURSOR::set_value`时C语言应用程序必须总是使用uint8_t类型（或者等价的unsigned char），传给`WT_CURSOR::get_value`的指针同样如此。如果一个位域的值与其它类型组合到一个包装格式，它等价于'B'，按一个完整的字节存储。

当通过记录号引用的时候（即键的格式为'r'），'t'类型存储在定长的列存中，不会有额外的值表示记录不存在。在这种情况下，零字节的值表示记录不存在。这意味着使用`WT_CURSOR::remove`删除一条记录等同于使用`WT_CURSOR::update`在记录中存储0值（在记录中存储零值会使得游标扫描时跳过这条记录）。另外，在对象结尾之后创建记录会隐式地使用为0值的字节创建任何缺失的中间记录。

'u'类型用于原始的字节数组：如果它出现在格式字符串的末尾（包括未类型化的表的默认'u'格式），那么大小不会显式存储。当'u'出现在格式字符串中间时，大小以32位整数按相同的字节序存储在格式字符串的最后，后面跟着数据。

有一个默认的比较器来按字典序比较，并且默认的编码经过设计，编码后的键的字典序通常情况下是预期的顺序。例如，经过设计，整数的可变长度编码在默认的比较器下有自然的整数顺序。

关于WiredTiger的打包格式的详细信息可以查看[打包与解包数据](303_packing_and_unpacking_data.md)。

WiredTiger同样可以通过实现WT_COLLATOR接口来使用定制的比较器扩展。

**键和值的格式**

每个表都有一个键的格式和值的格式，就像在**列类型**中描述的。这些类型是在创建表的时候通过传key_format和value_format键给`WT_SESSION::create`来配置的。

例如，一个简单的行存储的表，使用字符串作为键和值，可以像下面这样创建：
```c
ret = session->create(session, "table:mytable", "key_format=S,value_format=S");
```

一个简单的列存储的表，使用字符串作为值，可以像下面这样创建：
```c
ret = session->create(session, "table:mytable", "key_format=r,value_format=S");
```

**游标格式**

表的游标与表自身有相同的键格式。游标的键列通过`WT_CURSOR::set_key`设置，通过`WT_CURSOR::get_key`访问。`WT_CURSOR::set_key`与`printf`类似，使用一个按照在key_format中配置的键列顺序排列的值的列表。

例如，使用字符串作为键为一个行存的表设置键可以这样做：
```c
/* Set the cursor's string key. */
const char *key = "another key";
cursor->set_key(cursor, key);
```

例如，为一个列存的表设置键可以这样做：
```c
uint64_t recno = 37;    /* Set the cursor's record number key. */
cursor->set_key(cursor, recno);
```

更复杂的例子，为一个行存表设置组合键，key_format是"SiH"，可以这样做：
```c
/* Set the cursor's "SiH" format composite key. */
cursor->set_key(cursor, "first", (int32_t)5, (uint16_t)7);
```

键的值通过`WT_CURSOR::get_key`访问，与`scanf`类似，使用一个按相同顺序指向值的指针的列表：
```c
const char *key;    /* Get the cursor's string key. */
ret = cursor->get_key(cursor, &key);
```
```c
uint64_t recno;    /* Get the cursor's record number key. */
ret = cursor->get_key(cursor, &recno);
```
```c
/* Get the cursor's "SiH" format composite key. */
const char *first;
int32_t second;
uint16_t third;
ret = cursor->get_key(cursor, &first, &second, &third);
```

表的游标与表有相同的值格式，除非在`WT_SESSION::open_cursor`上配置了投影。查看[游标](203_cursors.md)中的投影以获取更多信息。

`WT_CURSOR::set_value`用来设置值列，而`WT_CURSOR::get_value`用来获取值列，与`WT_CURSOR::set_key`和`WT_CURSOR::get_key`的方式相同。

**列**

表的列可以通过传递columns键给`WT_SESSION::create`来赋予名字。列的名字首先赋给key_format中的列，然后是value_format中的列。每个列都必须有一个名字，并且列名不能重复。

例如，一个列存的表有一个employee ID作为键，3个列（department、salary和first year of employment），可以这样创建：
```c
/*
 * Create a table with columns: keys are record numbers, values are
 * (string, signed 32-bit integer, unsigned 16-bit integer).
 */
ret = session->create(session, "table:mytable",
                               "key_format=r, value_format=SiH",
                               "columns=(id,department,salary,year-started)");
```

在这个例子中，键的列名是id，值的列名是department, salary和year-started（id对应列格式r，department对应列的值格式S，salary对应值格式i，year-started对应值格式H）。

表一旦创建，在应用程序接下来的运行中就不必再调用`WT_SESSION::create`。然而，无论如何值得这样做，这样既能验证表已经存在，同时也能验证表的模式与应用程序预期的模式匹配。

**列组**

一旦列的名字被赋值，就可以用来配置列组。列组主要用来定义存储以便调优缓存的行为，因为每个列组存储在单独的文件中。

设置列组有两个步骤：首先，向`WT_SESSION::create`传递在colgroups配置键中的列组的名称列表。然后为每个列组调用`WT_SESSION::create`，使用URI `colgroup:<table>:<colgroup name>`和columns键。每个列必须出现在至少一个列组中，列可以在多个列组中，这时列会被存储在多个文件中。

例如，下面的数据被存储在WiredTiger的表中：
```c
/* The C struct for the data we are storing in a WiredTiger table. */
typedef struct {
    char country[5];
    uint16_t year;
    uint64_t population;
} POP_RECORD;

static POP_RECORD pop_data[] = {
    {"AU",  1900,   4000000},
    {"AU",  2000,  19053186},
    {"CAN", 1900,   5500000},
    {"CAN", 2000,  31099561},
    {"UK",  1900, 369000000},
    {"UK",  2000,  59522468},
    {"USA", 1900,  76212168},
    {"USA", 2000, 301279593},
    {"", 0, 0}
};
```

如果我们主要想访问人口信息，但是在访问其它信息时也想访问人口信息，我们可以把所有的列存储在一个文件中，并在另一个文件中存储人口列的一个额外的副本：
```c
/*
 * Create the population table.
 * Keys are record numbers, the format for values is (5-byte string,
 * uint16_t, uint64_t).
 * See: wiredtiger_struct_pack for details of the format strings.
 */
ret = session->create(session, "table:poptable",
                               "key_format=r,"
                               "value_format=5sHQ,"
                               "columns=(id,country,year,population),"
                               "colgroups=(main,population)");

/*
 * Create two column groups: a primary column group with the country
 * code, year and population (named "main"), and a population column
 * group with the population by itself (named "population").
 */
ret = session->create(session, "colgroup:poptable:main", "columns=(country,year,population)");
ret = session->create(session, "colgroup:poptable:population", "columns=(population)");
```

列组总是与表有相同的键。这对于列存特别有用，因为记录号不显式存储在磁盘上，因此在多个文件上没有重复的键。在行存列组上键会复制到多个文件上。

游标可以在列组上打开，通过传递列组的URI给`WT_SESSION::open_cursor`方法。例如，人口可以同时从我们创建的两个列组中搜索：
```c
/*
 * Open a cursor on the main column group, and return the information
 * for a particular country.
 */
ret = session->open_cursor(session, "colgroup:poptable:main", NULL, NULL, &cursor);
cursor->set_key(cursor, 2);
if ((ret = cursor->search(cursor)) == 0) {
    ret = cursor->get_value(cursor, &country, &year, &population);
    printf("ID 2: country %s, year %u, population %" PRIu64 "\n", country, year, population);
}
```
```c
/*
 * Open a cursor on the population column group, and return the
 * population of a particular country.
 */
ret = session->open_cursor(session, "colgroup:poptable:population", NULL, NULL, &cursor);
cursor->set_key(cursor, 2);
if ((ret = cursor->search(cursor)) == 0) {
    ret = cursor->get_value(cursor, &population);
    printf("ID 2: population %" PRIu64 "\n", population);
}
```

键列可以包含在列组的列中。因为列组总是与表有相同的键，使用`WT_CURSOR::get_key`搜索列组的键列，而不是`WT_CURSOR::get_value`。

**索引**

在表上列也被用于创建和配置索引。

当表被修改时表的索引自动更新。

表的索引游标是只读的，不能用于更新操作。

要创建表的索引，调用`WT_SESSION::create`，使用URI `index:<table>:<index name>`，在配置中给出列的列表。

继续前面的例子，我们可以在country列上打开索引：
```c
/* Create an index with a simple key. */
ret = session->create(session, "index:poptable:country", "columns=(country)");
```

通过为`WT_SESSION::open_cursor`方法传递索引的URI在索引上打开游标。

对于`WT_CURSOR::get_key`和`WT_CURSOR::set_key`，索引游标使用指定的索引的键列。例如，可以从country索引搜索信息：
```c
/* Search in a simple index. */
ret = session->open_cursor(session, "index:poptable:country", NULL, NULL, &cursor);
cursor->set_key(cursor, "AU\0\0\0");
ret = cursor->search(cursor);
ret = cursor->get_value(cursor, &country, &year, &population);
printf("AU: country %s, year %u, population %" PRIu64 "\n", country, (unsigned int)year, population);
```

要使用组合键创建索引，在调用`WT_SESSION::create`时指定多于一个列：
```c
/* Create an index with a composite key (country, year). */
ret = session->create(session, "index:poptable:country_plus_year", "columns=(country, year)");
```

要从组合索引上查找信息需要更复杂的`WT_CURSOR::set_key`调用，但是其它方面都一样：
```c
/* Search in a composite index. */
ret = session->open_cursor(session, "index:poptable:country_plus_year", NULL, NULL, &cursor);
cursor->set_key(cursor, "USA\0\0", (uint16_t)1900);
ret = cursor->search(cursor);
ret = cursor->get_value(cursor, &country, &year, &population);
printf("US 1900: country %s, year %u, population %" PRIu64 "\n", country, (unsigned int)year, population);
```

**不可变索引**

可以创建一个使用immutable配置项的索引。这个设置告诉WiredTiger在记录更新时不改变记录的索引键。这是一个优化，节省了删除和插入索引，无论主表中的值何时被更新。

如果配置了不可变，当更新应该改变索引的内容时可能会导致数据冲突。

使用不可变索引的例子：
```c
/* Create an immutable index. */
ret = session->create(session, "index:poptable:immutable_year", "columns=(year), immutable");
```

**索引游标投影**

默认情况下，使用`WT_CURSOR::get_value`时索引游标返回表的所有值列。应用程序在调用`WT_CURSOR::get_value`时可以指定应该返回的列，通过附加一个列的列表到`WT_SESSION::open_cursor`的uri参数。这被称为投影，查看[投影]()以获取详细信息。

在索引游标上，使用投影可以避免在不包含与操作相关的列的列组上查询。

下面的例子只从索引返回表的主键（记录号）：
```c
/*
 * Use a projection to return just the table's record number key
 * from an index.
 */
ret = session->open_cursor(session, "index:poptable:country_plus_year(id)", NULL, NULL, &cursor);
while ((ret = cursor->next(cursor)) == 0) {
    ret = cursor->get_key(cursor, &country, &year);
    ret = cursor->get_value(cursor, &recno);
    printf("row ID %" PRIu64 ": country %s, year %u\n", recno, country, year);
}
```

下面投影的例子从索引返回列的子集：
```c
/*
 * Use a project to return just the population column from an
 * index.
 */
ret = session->open_cursor(session, "index:poptable:country_plus_year(population)", NULL, NULL, &cursor);
while ((ret = cursor->next(cursor)) == 0) {
    ret = cursor->get_key(cursor, &country, &year);
    ret = cursor->get_value(cursor, &population);
    printf("population %" PRIu64 ": country %s, year %u\n", population, country, year);
```

出于性能的原因，在索引上的性能关键的操作上包含所有的列是值得的，这样就可能执行只使用索引的查找，表上的列组不会被访问。这里，所有的“热”列应该包括在索引中（总是把“真正”的索引键列放在前面，这会决定排序顺序）。然后，在索引上打开游标不会返回任何值列，并且没有列组会被访问。
```c
/*
 * Use a projection to avoid accessing any other column groups when
 * using an index: supply an empty list of value columns.
 */
ret = session->open_cursor(session, "index:poptable:country_plus_year()", NULL, NULL, &cursor);
while ((ret = cursor->next(cursor)) == 0) {
    ret = cursor->get_key(cursor, &country, &year);
    printf("country %s, year %u\n", country, year);
}
```

可以不使用记录号作为索引键创建列存对象的索引游标（列存上索引键是记录号的二级索引没有用处）

**示例代码**

上面的代码是从完整的示例程序ex_schema.c中摘取的。

下面是另一个示例程序ex_call_center.c。
```c
/*
 * In SQL, the tables are described as follows:
 *
 * CREATE TABLE Customers(id INTEGER PRIMARY KEY,
 *     name VARCHAR(30), address VARCHAR(50), phone VARCHAR(15))
 * CREATE INDEX CustomersPhone ON Customers(phone)
 *
 * CREATE TABLE Calls(id INTEGER PRIMARY KEY, call_date DATE,
 *     cust_id INTEGER, emp_id INTEGER, call_type VARCHAR(12),
 *     notes VARCHAR(25))
 * CREATE INDEX CallsCustDate ON Calls(cust_id, call_date)
 *
 * In this example, both tables will use record numbers for their IDs, which
 * will be the key.  The C structs for the records are as follows.
 */
/* Customer records. */
typedef struct {
        uint64_t id;
        const char *name;
        const char *address;
        const char *phone;
} CUSTOMER;
/* Call records. */
typedef struct {
        uint64_t id;
        uint64_t call_date;
        uint64_t cust_id;
        uint64_t emp_id;
        const char *call_type;
        const char *notes;
} CALL;
```
```c
        ret = conn->open_session(conn, NULL, NULL, &session);
        /*
         * Create the customers table, give names and types to the columns.
         * The columns will be stored in two groups: "main" and "address",
         * created below.
         */
        ret = session->create(session, "table:customers",
            "key_format=r,"
            "value_format=SSS,"
            "columns=(id,name,address,phone),"
            "colgroups=(main,address)");
        /* Create the main column group with value columns except address. */
        ret = session->create(session,
            "colgroup:customers:main", "columns=(name,phone)");
        /* Create the address column group with just the address. */
        ret = session->create(session,
            "colgroup:customers:address", "columns=(address)");
        /* Create an index on the customer table by phone number. */
        ret = session->create(session,
            "index:customers:phone", "columns=(phone)");
        /* Populate the customers table with some data. */
        ret = session->open_cursor(
            session, "table:customers", NULL, "append", &cursor);
        for (custp = cust_sample; custp->name != NULL; custp++) {
                cursor->set_value(cursor,
                    custp->name, custp->address, custp->phone);
                ret = cursor->insert(cursor);
        }
        ret = cursor->close(cursor);
        /*
         * Create the calls table, give names and types to the columns.  All the
         * columns will be stored together, so no column groups are declared.
         */
        ret = session->create(session, "table:calls",
            "key_format=r,"
            "value_format=qrrSS,"
            "columns=(id,call_date,cust_id,emp_id,call_type,notes)");
        /*
         * Create an index on the calls table with a composite key of cust_id
         * and call_date.
         */
        ret = session->create(session, "index:calls:cust_date",
            "columns=(cust_id,call_date)");
        /* Populate the calls table with some data. */
        ret = session->open_cursor(
            session, "table:calls", NULL, "append", &cursor);
        for (callp = call_sample; callp->call_type != NULL; callp++) {
                cursor->set_value(cursor, callp->call_date, callp->cust_id,
                    callp->emp_id, callp->call_type, callp->notes);
                ret = cursor->insert(cursor);
        }
        ret = cursor->close(cursor);
        /*
         * First query: a call arrives.  In SQL:
         *
         * SELECT id, name FROM Customers WHERE phone=?
         *
         * Use the cust_phone index, lookup by phone number to fill the
         * customer record.  The cursor will have a key format of "S" for a
         * string because the cust_phone index has a single column ("phone"),
         * which is of type "S".
         *
         * Specify the columns we want: the customer ID and the name.  This
         * means the cursor's value format will be "rS".
         */
        ret = session->open_cursor(session,
            "index:customers:phone(id,name)", NULL, NULL, &cursor);
        cursor->set_key(cursor, "123-456-7890");
        ret = cursor->search(cursor);
        if (ret == 0) {
                ret = cursor->get_value(cursor, &cust.id, &cust.name);
                printf("Read customer record for %s (ID %" PRIu64 ")\n",
                    cust.name, cust.id);
        }
        ret = cursor->close(cursor);
        /*
         * Next query: get the recent order history.  In SQL:
         *
         * SELECT * FROM Calls WHERE cust_id=? ORDER BY call_date DESC LIMIT 3
         *
         * Use the call_cust_date index to find the matching calls.  Since it is
         * is in increasing order by date for a given customer, we want to start
         * with the last record for the customer and work backwards.
         *
         * Specify a subset of columns to be returned.  (Note that if these were
         * all covered by the index, the primary would not have to be accessed.)
         * Stop after getting 3 records.
         */
        ret = session->open_cursor(session,
            "index:calls:cust_date(cust_id,call_type,notes)",
            NULL, NULL, &cursor);
        /*
         * The keys in the index are (cust_id,call_date) -- we want the largest
         * call date for a given cust_id.  Search for (cust_id+1,0), then work
         * backwards.
         */
        cust.id = 1;
        cursor->set_key(cursor, cust.id + 1, 0);
        ret = cursor->search_near(cursor, &exact);
        /*
         * If the table is empty, search_near will return WT_NOTFOUND, else the
         * cursor will be positioned on a matching key if one exists, or an
         * adjacent key if one does not.  If the positioned key is equal to or
         * larger than the search key, go back one.
         */
        if (ret == 0 && exact >= 0)
                ret = cursor->prev(cursor);
        for (count = 0; ret == 0 && count < 3; ++count) {
                ret = cursor->get_value(cursor,
                    &call.cust_id, &call.call_type, &call.notes);
                if (call.cust_id != cust.id)
                        break;
                printf("Call record: customer %" PRIu64 " (%s: %s)\n",
                    call.cust_id, call.call_type, call.notes);
                ret = cursor->prev(cursor);
        }
```
