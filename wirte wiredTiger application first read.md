# Write wiredtiger application

This section covers topics of interest for programmers writing WiredTiger applications.
本部分介绍了开发者使用wiredtiger开发应用的常用的部分。

We follow SQL terminology: a database is set of tables managed together. Tables consist of rows, where each row is a key and its associated value. Tables may optionally have an associated schema, splitting the value into a set of columns. Tables may also have associated indices, each of which is ordered by one or more columns.

使用的SQL相关的术语如下：
            database：set of tables managed together
            tables：consist of rows
            row：key = associated value

table可以有相关的schema，把value分到不同的列， table也可能有indices，其中的每一个是按照某一列或者多列排序。

## 使用API

### get start

WiredTiger applications 使用如下的类进行数据的管理和获取：

- a **WT_CONNECTION** represents a connection to a database. Most applications will only open one connection to a database for each process. All methods in **WT_CONNECTION** are thread safe.
（**WT_CONNECTION** 表示一个数据库的连接，大多数应用程序只会为每个进程打开一个与数据库的连接，所有在 **WT_CONNECTION** 里面的方法都是线程安全的。）

- a **WT_SESSION** represents a context in which database operations are performed. Sessions are opened on a specified connection, and applications must open a single session for each thread accessing the database.
（**WT_SESSION** 表示执行数据库操作的上下文背景，在指定的连接上打开会话，并且应用程序必须为访问数据库的每个线程打开一个会话。）

- a **WT_CURSOR** represents a cursor over a collection of data. Cursors are opened in the context of a session (which may have an associated transaction), and can query and update records. In the common case, a cursor is used to access records in a table. However, cursors can be used on subsets of tables (such as a single column or a projection of multiple columns), as an interface to statistics, configuration data or application-specific data sources.
（**WT_CURSOR** 表示数据集合上的游标，游标在会话（可能具有事务）的上下文中打开，并且可以查询和更新记录。 在通常情况下，游标用于访问表中的记录。 然而，光标可以在表（诸如单个列或多个列的投影）的子集被使用，如作为一个统计数据的接口，配置数据或应用程序的特定的数据源。）

使用strings 进行 对应的handle和operation的配置。WiredTiger supports the C, C++, Java and Python programming languages (among others).

WiredTiger 默认工作方式为传统的k、v存储，其中key和value都是raw byte arrays，使用**WT_ITEM** 数据结构进行获取。key和value的大小可能范围为（4GB-512B），取决于**WT_SESSION** 创建时的"maximum item sizes"的参数的配置，大的key和value将存储在溢出页中。

WiredTiger还支持schema层，以便可以从列表中选择键和值类型，也可以从任何类型组合形成的列组成的复合键或值中进行选择。 键和值的大小（4GB-512B）字节限制仍然适用。

使用WiredTiger的所有应用程序的结构大致如下。 下面的代码来自完整的示例程序ex_access.c。

#### Connecting to a database


要访问数据库，首先打开一个连接并为访问数据库的单线程创建会话句柄：

```cpp
WT_CONNECTION *conn;
WT_CURSOR *cursor;
WT_SESSION *session;
const char *key, *value;
int ret;
/* Open a connection to the database, creating it if necessary. */
error_check(wiredtiger_open(home, NULL, "create", &conn));
//The configuration string "create" is passed to wiredtiger_open to indicate the database should be created if it does not already exist.
/* Open a session handle for the database. */
error_check(conn->open_session(conn, NULL, NULL, &session));

//The code block above also shows simple error handling with wiredtiger_strerror (a function that returns a string describing an error code passed as its argument). More complex error handling can be configured by passing an implementation of WT_EVENT_HANDLER to wiredtiger_open or WT_CONNECTION::open_session.
```

#### Creating a table

Create a table we can use to store data:

```cpp
error_check(session->create(session, "table:access", "key_format=S,value_format=S"));
 //This call creates a table called "access", configured to use strings for its key and value columns. (See Schema, Columns, Column Groups, Indices and Projections for more information on tables with other types of key and value columns.)
```

#### Accessing data with cursors

Now that we have a table, we open a cursor to perform some operations on it:

```cpp
error_check(session->open_cursor(session, "table:access", NULL, NULL, &cursor));
//the string "table:access" specifies that we are opening the cursor on the table named "access".

cursor->set_key(cursor, "key1"); /* Insert a record. */
cursor->set_value(cursor, "value1");
error_check(cursor->insert(cursor));
// insert a new row into the table. The WT_CURSOR::set_key and WT_CURSOR::set_value calls put the application's key and value into the cursor, respectively. The WT_CURSOR::insert call creates a record containing that value and inserts it into the table.

error_check(cursor->reset(cursor)); /* Restart the scan. */
//Because the cursor was positioned in the table after the WT_CURSOR::insert call, we had to re-position it using the WT_CURSOR::reset call;

while ((ret = cursor->next(cursor)) == 0) {
error_check(cursor->get_key(cursor, &key));
error_check(cursor->get_value(cursor, &value));
printf("Got record: %s : %s\n", key, value);
}
scan_end_check(ret == WT_NOTFOUND); /* Check for end-of-table. */
// iterate through all of the records in the table, printing them out as we go:


```

请注意，记录的键和值部分以C字符串形式返回，因为该表是通过这种方式创建的（即使它是由示例的上一次运行创建的）。 应用程序中不需要数据提取或转换。

if we weren't using the cursor for the call to WT_CURSOR::insert above, this loop would simplify to:

```cpp
while ((ret = cursor->next(cursor)) == 0) {
    ...
}
```

#### Closing handles

Lastly, we close the connection, which implicitly closes the cursor and session handles:

```cpp
error_check(conn->close(conn, NULL)); /* Close all handles. */
```


### configure strings

WiredTiger中的方法采用配置字符串来提供可选参数并配置非标准行为。 这些字符串是简单的逗号分隔列表，其中包含"<key> = <value>" 对，并且都具有相同的格式：

```
[key['='value]][','[key['='value]]]*
```

可以直接指定键和值以及由字母数字字符组成的值。 更确切地说，符合此正则表达式的键或值不需要引号：

```
 [-_0-9A-Za-z./][^\t\r\n :=,\])}]*
```

Configuration strings are **case-sensitive**

更复杂的键和值可以用双引号引用它们来指定。

例如，在打开与数据库的连接时使用配置字符串来指定是否应创建数据库并设置缓存大小：


```cpp
error_check(wiredtiger_open(
  home, NULL, "create,cache_size=5GB,log=(enabled,recover=on),statistics=(all)", &conn));
```

Configuration strings are also used to configure the table schema. For example, to create a table that uses C language strings for keys and values:
配置string也用于配置表的模式，如，创建一个表，使用C语言式的string指定key和value。

```cpp
error_check(session->create(session, "table:mytable", "key_format=S,value_format=S"));
```

为了处理更复杂的schema配置（例如在表中指定多个列），使用括号将值嵌套。 例如：

```cpp
/*
 * Create a table with columns: keys are record numbers, values are
 * (string, signed 32-bit integer, unsigned 16-bit integer).
 */
error_check(session->create(session, "table:mytable",
"key_format=r,value_format=SiH,"
"columns=(id,department,salary,year-started)"));
```

所有类型的括号是由解析器等效处理。

当期望一个整数值时，该值可以附加多个字符，如下所示：

Character   Meaning Change to value
B or b  byte    no change
K or k  kilobyte    multiply by 2^10
M or m  megabyte    multiply by 2^20
G or g  gigabyte    multiply by 2^30
T or t  terabyte    multiply by 2^40
P or p  petabyte    multiply by 2^50

For example, the value 500B is the same as entering the number 500, the value 500K is the same as 512000 and the string 500GB is the same as 536870912000.

"boolean"类型的值可以取为：false， true，0 or 1. 如果没有指定一个值，默认为1. 如下的配置中'overwrite'参数的值是相同的意思

```cpp
error_check(session->open_cursor(session, "table:mytable", NULL, "overwrite", &cursor));
error_check(
session->open_cursor(session, "table:mytable", NULL, "overwrite=true", &cursor));
error_check(session->open_cursor(session, "table:mytable", NULL, "overwrite=1", &cursor));
```

配置strings 是按照从左到右的顺序处理的，后面的配置将会覆盖前面的配置（除非该方法支持键的多个设置），

Configuration strings are processed in order from left to right, with later settings overriding earlier ones (unless multiple settings for a key are supported by the method).

配置字符串中多余的逗号和空格将被忽略（包括字符串的开头和结尾），因此始终可以安全地通过将两个配置字符串连接起来并以逗号之间的方式将它们组合在一起。

在C或C ++中，空的配置字符串可以通过传递NULL来用表示表示。


#### JavaScript Object Notation (JSON) compatibility

WiredTiger配置字符串与JavaScript Object Notation（JSON）兼容，并且将接受以下格式：

- parentheses may be round or square brackets or curly braces: '()', '[]' or '{}'

- the whole configuration string may optionally be wrapped in parentheses

- the key/value separator can be a colon: ':'

- keys and values may be in double quotes: "key" = "value"

- quoted strings are interpreted as UTF-8 values

结果是，无论需要什么配置字符串，应用程序都可以传递表示有效JSON对象的字符串。

由于JSON兼容性允许将冒号用作键/值分隔符，因此WiredTiger URI可能需要引用。 例如，以下WT_SESSION :: checkpoint调用将一组URI指定为检查点目标，使用双引号字符来防止解析器将冒号字符视为JSON名称/值分隔符：

```cpp
 /*
 * Checkpoint a list of objects. JSON parsing requires quoting the list of target URIs.
 */
error_check(session->checkpoint(session, "target=(\"table:table1\",\"table:table2\")"));
```


### Curors

Wiredtiger中常用的操作一般使用**WT_CURSOR**句柄，A cursor includes:

- a position within a data source
- getter/setters for key and value fields
- encoding of fields to store in the data source
- methods to navigate within and iterate through the data

Cursor operations 部分描述了更多如何使用 cursors.

#### Cursor types

内建的Cursor types：

- table:<table name>[<projection>] ,    table cursor ,  table key, table value(s) with optional projection of columns

- colgroup:<table name>:<column group name>  ,  column group cursor ,   table key, column group value(s) 

- index:<table name>:<index name>[<projection>] ,   index cursor    , key=index key, value=table value(s) with optional projection of columns 

- join:table:<table name>[<projection>] ,   join cursor ,   key=table key, value=table value(s) with optional projection of columns

可以使用以下特殊的游标类型来完成某些管理任务，这些游标类型可以访问WiredTiger管理的数据：

- backup:   backup cursor;  key=string, see Backups for details

- log:  log cursor ;    key=(long fileID, long offset, int seqno),
value=(uint64_t txnid, uint32_t rectype,uint32_t optype, uint32_t fileid,WT_ITEM key, WT_ITEM value),
see Log cursors for details

- metadata:[create] ; metadata cursor (optionally only returning configuration strings for WT_SESSION::create if create is appended ;key=string, value=string, see Reading WiredTiger Metadata for details

- statistics:[<data source URI>] ;  database    data source join or session statistics cursor ;     key=int id,
value=(string description, string value, uint64_t value), see Statistics Data for details

高级应用程序还可以打开以下低级游标类型：

- file:<file name>  ; file cursor;  file key, file value(s)
- lsm:<name>    ; LSM cursor ; (key=LSM key, value=LSM value)   LSM key, LSM value, see Log-Structured Merge Trees


See the following for more details:

- Data Sources
- Reading WiredTiger Metadata
- Log cursors
- Join cursors

#### Projections


表和索引上的游标可以返回列的子集。 这是通过在WT_SESSION :: open_cursor的uri参数中的括号中列出列名来实现。 WT_CURSOR :: get_value仅返回列出列中的字段。

ex_schema.c示例创建一个表，其值格式为“ 5sHq”，country为string，year为short， population 为 long，下面的列子仅返回表中的country 列和year列：

```cpp
   /*
 * Use a projection to return just the table's country and year columns.
 */
error_check(session->open_cursor(session, "table:poptable(country,year)", NULL, NULL, &cursor));
while ((ret = cursor->next(cursor)) == 0) {
error_check(cursor->get_value(cursor, &country, &year));
printf("country %s, year %" PRIu16 "\n", country, year);
}
```

这对于索引游标特别有用，因为如果投影中的所有列都在索引中可用（包括主键列，它们是索引的值），则可以从索引中读取数据而无需访问任何列组。 有关更多信息，请参见索引光标投影。

#### Cursors and Transactions

如果会话中有活动的事务，则游标将在该事务的上下文中操作。 事务处于活动状态时执行的读取将继承事务的隔离级别，并且通过调用WT_SESSION :: commit_transaction使在事务内执行的更新持久，或者通过调用WT_SESSION :: rollback_transaction放弃该更新。

If no transaction is active, cursor reads are performed at the isolation level of the session, set with the isolation configuration key to WT_CONNECTION::open_session and successful updates are automatically committed before the update operation completes.

如果没有事务处于活动状态，则在会话的隔离级别执行游标读取，并使用隔离配置键将其设置为WT_CONNECTION :: open_session，并在更新操作完成之前自动提交成功的更新。

Any operation that consists of multiple related updates should be enclosed in an explicit transaction to ensure that the updates are applied atomically.

任何包含多个相关更新的操作都应包含在显式事务中，以确保自动应用更新。

At read-committed (the default) or snapshot isolation levels, committed changes from concurrent transactions become visible when no cursor is positioned. In other words, at these isolation levels, all cursors in a session read from a stable snapshot while any cursor in the session remains positioned. A call to WT_CURSOR::next or WT_CURSOR::prev on a positioned cursor will not update the snapshot.

在读提交（默认）或快照隔离级别，当未定位游标时，来自并发事务的已提交更改将变得可见。 换句话说，在这些隔离级别上，当会话中的任何游标仍处于定位状态时，会话中的所有游标都从稳定的快照读取。 在定位的游标上调用WT_CURSOR :: next或WT_CURSOR :: prev不会更新快照。


Cursor positions survive transaction boundaries, unless a transaction is rolled back. When a transaction is rolled-back either implicitly or explicitly, all cursors in the session are reset as if the WT_CURSOR::reset method was called, discarding any cursor position as well as any key and value.

除非事务回滚，否则游标位置会保留在事务的边界之内。 当隐式或显式回滚事务时，将重置会话中的所有游标，就像调用WT_CURSOR :: reset方法一样，丢弃任何游标位置以及任何键和值。

See Transactions for more information.

#### Cursors and Eviction

Cursor positions hold resources that can inhibit the eviction of memory pages. If a cursor is active on a page being considered for eviction, the eviction will defer until the cursor is moved or reset. To avoid this and to keep resources freed in general, an application should call WT_CURSOR::reset during times it does not need to keep the cursor positioned. A cursor that has been reset is not active and will not inhibit eviction.


游标位置保存的资源可能会阻止内存页的淘汰。 如果光标在要淘汰的页面上处于活动状态，则内存页淘汰将推迟到光标移动或重置为止。 为了避免这种情况并总体上释放资源，应用程序应在不需要保持光标定位期间调用WT_CURSOR :: reset。 已重置的游标不处于活动状态，并且不会禁止内存页淘汰。

#### Raw mode

Cursors can be configured for raw mode by specifying the "raw" config keyword to WT_SESSION::open_cursor. In this mode, the methods WT_CURSOR::get_key, WT_CURSOR::get_value, WT_CURSOR::set_key and WT_CURSOR::set_value all take a single WT_ITEM in the variable-length argument list instead of a separate argument for each column.

WT_ITEM structures do not need to be cleared before use.

For WT_CURSOR::get_key and WT_CURSOR::get_value in raw mode, the WT_ITEM can be split into columns by calling WT_EXTENSION_API::struct_unpack with the cursor's key_format or value_format, respectively. For WT_CURSOR::set_key and WT_CURSOR::set_value in raw mode, the WT_ITEM should be equivalent to calling WT_EXTENSION_API::struct_pack for the cursor's key_format or value_format, respectively.

The ex_schema.c example creates a table where the value format is "5sHq", where the initial string is the country, the short is a year, and the long is a population. The following example lists the table record values, using raw mode:

```cpp
/* List the records in the table using raw mode. */
error_check(session->open_cursor(session, "table:poptable", NULL, "raw", &cursor));
while ((ret = cursor->next(cursor)) == 0) {
    WT_ITEM key, value;
error_check(cursor->get_key(cursor, &key));
error_check(wiredtiger_struct_unpack(session, key.data, key.size, "r", &recno));
printf("ID %" PRIu64, recno);
error_check(cursor->get_value(cursor, &value));
error_check(wiredtiger_struct_unpack(
      session, value.data, value.size, "5sHQ", &country, &year, &population));
printf(
": country %s, year %" PRIu16 ", population %" PRIu64 "\n", country, year, population);
}
scan_end_check(ret == WT_NOTFOUND);
```

Raw mode can be used in combination with projections. The following example lists just the country and year columns from the table record values, using raw mode:

```cpp
  /*
 * Use a projection to return just the table's country and year columns, using raw mode.
 */
error_check(
session->open_cursor(session, "table:poptable(country,year)", NULL, "raw", &cursor));
while ((ret = cursor->next(cursor)) == 0) {
    WT_ITEM value;
error_check(cursor->get_value(cursor, &value));
error_check(
wiredtiger_struct_unpack(session, value.data, value.size, "5sH", &country, &year));
printf("country %s, year %" PRIu16 "\n", country, year);
}
scan_end_check(ret == WT_NOTFOUND);
```

#### Reading WiredTiger Metadata


WiredTiger游标提供对来自各种来源的数据的访问。 其中之一是数据库中的对象列表

要检索数据库对象列表，请在uri metadata上打开一个光标：。 每个返回的键将是一个数据库对象，每个返回的值将是存储在该键所命名的对象的元数据中的信息。


For example:

```cpp
error_check(session->open_cursor(session, "metadata:", NULL, NULL, &cursor));
```

To retrieve value strings that are valid arguments for calls to WT_SESSION::create, open a cursor on metadata:create.

The metadata cursor is read-only, and the metadata cannot be modified.


### transaction

#### ACID properties

Transactions provide a powerful abstraction for multiple threads to operate on data concurrently because they have the following properties:

- Atomicity: all or none of a transaction is completed.

- Consistency: if each transaction maintains some property when considered separately, then the combined effect of executing the transactions concurrently will maintain the same property.

- Isolation: developers can reason about transactions as if they run single-threaded.

- Durability: once a transaction commits, its updates cannot be lost.

WiredTiger 通过下面的方式支持事务的ACID的特性：

- the maximum level of isolation supported is snapshot isolation. See Isolation levels for more details.

- transactional updates are made durable by a combination of checkpoints and logging. See Checkpoint durability for information on checkpoint durability and Commit-level durability for information on commit-level durability.

- each transaction's uncommitted changes must fit in memory: for efficiency, WiredTiger does not write to the log until a transaction commits.

#### Transactional API

在WiredTiger中，事务操作是WT_SESSION类的方法。

Applications call **WT_SESSION::begin_transaction** to start a new transaction. Operations subsequently performed using that WT_SESSION handle, including operations on any cursors open in that **WT_SESSION** handle (whether opened before or after the **WT_SESSION::begin_transaction** call), are part of the transaction and their effects committed by calling **WT_SESSION::commit_transaction**, or discarded by calling **WT_SESSION::rollback_transaction**. Applications that use Application-specified Transaction Timestamps can utilize the **WT_SESSION::prepare_transaction** API as a basis for implementing a two phase commit protocol.

If **WT_SESSION::commit_transaction** returns an error for any reason, the transaction was rolled back, not committed.

When transactions are used, data operations can encounter a conflict and fail with the WT_ROLLBACK error. If this error occurs, transactions should be rolled back with **WT_SESSION::rollback_transaction** and the operation retried.

The **WT_SESSION::rollback_transaction** method implicitly resets all cursors in the session as if the **WT_CURSOR::reset** method was called, discarding any cursor position as well as any key and value.

```cpp
   /*
 * Cursors may be opened before or after the transaction begins, and in either case, subsequent
 * operations are included in the transaction. Opening cursors before the transaction begins
 * allows applications to cache cursors and use them for multiple operations.
 */
error_check(session->open_cursor(session, "table:mytable", NULL, NULL, &cursor));
error_check(session->begin_transaction(session, NULL));
cursor->set_key(cursor, "key");
cursor->set_value(cursor, "value");
switch (cursor->update(cursor)) {
case 0: /* Update success */
error_check(session->commit_transaction(session, NULL));
    /*
     * If commit_transaction succeeds, cursors remain positioned; if commit_transaction fails,
     * the transaction was rolled-back and all cursors are reset.
     */
break;
case WT_ROLLBACK: /* Update conflict */
default:          /* Other error */
error_check(session->rollback_transaction(session, NULL));
    /* The rollback_transaction call resets all cursors. */
break;
}
/*
 * Cursors remain open and may be used for multiple transactions.
 */
```


#### Implicit transactions

If a cursor is used when no explicit transaction is active in a session, reads are performed at the isolation level of the session, set with the isolation key to WT_CONNECTION::open_session, and successful updates are automatically committed before the update operation returns.

如果在会话中没有显式事务处于活动状态时使用游标，则会在会话的隔离级别执行读取，并使用隔离键设置为WT_CONNECTION :: open_session，并在更新操作返回之前自动提交成功的更新。


Any operation consisting of multiple related updates should be enclosed in an explicit transaction to ensure the updates are applied atomically.

包含多个相关更新的任何操作都应包含在显式事务中，以确保自动应用更新。


If an implicit transaction successfully commits, the cursors in the WT_SESSION remain positioned. If an implicit transaction fails, all cursors in the WT_SESSION are reset, as if WT_CURSOR::reset were called, discarding any position or key/value information they may have.

如果隐式事务成功提交，则WT_SESSION中的游标将保持定位。 如果隐式事务失败，则将重置WT_SESSION中的所有游标，就像调用了WT_CURSOR :: reset一样，丢弃它们可能具有的任何位置或键/值信息。

See Cursors and Transactions for more information.

#### Concurrency control

WiredTiger uses optimistic concurrency control algorithms. This avoids the bottleneck of a centralized lock manager and ensures transactional operations do not block: reads do not block writes, and vice versa.

WiredTiger使用乐观并发控制算法。 这样可以避免集中式锁管理器的瓶颈，并确保不会阻塞事务操作：读取不会阻塞写入，反之亦然。

Further, writes do not block writes, although concurrent transactions updating the same value will fail with WT_ROLLBACK. Some applications may benefit from application-level synchronization to avoid repeated attempts to rollback and update the same value.

此外，写操作不会阻止写操作，尽管更新相同值的并发事务将因WT_ROLLBACK而失败。 某些应用程序可能会受益于应用程序级同步，以避免重复尝试回滚和更新相同的值。


Operations in transactions may also fail with the WT_ROLLBACK error if some resource cannot be allocated after repeated attempts. For example, if the cache is not large enough to hold the updates required to satisfy transactional readers, an operation may fail and return WT_ROLLBACK.

如果在重复尝试后无法分配某些资源，则事务中的操作也可能因WT_ROLLBACK错误而失败。 例如，如果缓存不足以容纳满足事务性读取器所需的更新，则操作可能会失败并返回WT_ROLLBACK。

#### Isolation levels

WiredTiger supports **read-uncommitted**, **read-committed** and **snapshot** isolation levels; the **default isolation** level is read-committed.

- **read-uncommitted**: Transactions can see changes made by other transactions before those transactions are committed. Dirty reads, non-repeatable reads and phantoms are possible.

- **read-committed**: Transactions cannot see changes made by other transactions before those transactions are committed. Dirty reads are not possible; non-repeatable reads and phantoms are possible. Committed changes from concurrent transactions become visible when no cursor is positioned in the read-committed transaction.

- **snapshot**: Transactions read the versions of records committed before the transaction started. Dirty reads and non-repeatable reads are not possible; phantoms(幻读) are possible.

Snapshot isolation is a strong guarantee, but not equivalent to a single-threaded execution of the transactions, known as serializable isolation. Concurrent transactions T1 and T2 running under snapshot isolation may both commit and produce a state that neither (T1 followed by T2) nor (T2 followed by T1) could have produced, if there is overlap between T1's reads and T2's writes, and between T1's writes and T2's reads.

快照隔离是一个有力的保证，但不等同于事务的单线程执行，即可序列化隔离。 在T1的读取和T2的写入之间以及T1的写入和T2的读取之间存在重叠的情况下，在快照隔离下运行的并发事务T1和T2都可能提交并产生既不会产生（T1跟着T2）也没有产生（T2跟着T1）的状态。

可以基于每个事务配置事务隔离级别：

```cpp
/* A single transaction configured for snapshot isolation. */
error_check(session->open_cursor(session, "table:mytable", NULL, NULL, &cursor));
error_check(session->begin_transaction(session, "isolation=snapshot"));
cursor->set_key(cursor, "some-key");
cursor->set_value(cursor, "some-value");
error_check(cursor->update(cursor));
error_check(session->commit_transaction(session, NULL));
```

此外，可以在每个会话的基础上配置和重新配置默认事务隔离：

```cpp
   /* Open a session configured for read-uncommitted isolation. */
error_check(conn->open_session(conn, NULL, "isolation=read-uncommitted", &session));

 /* Re-configure a session for snapshot isolation. */
error_check(session->reconfigure(session, "isolation=snapshot"));
```

#### Named Snapshots

Applications can create named snapshots by calling WT_SESSION::snapshot with a configuration that includes "name=foo". This configuration creates a new named snapshot, as if a snapshot isolation transaction were started at the time of the WT_SESSION::snapshot call.

应用程序可以通过使用包含“ name = foo”的配置调用WT_SESSION :: snapshot来创建命名快照。 此配置将创建一个新的命名快照，就像在WT_SESSION :: snapshot调用时启动了快照隔离事务一样。

Subsequent transactions can be started "as of" that snapshot by calling WT_SESSION::begin_transaction with a configuration that includes snapshot=foo. That transaction will run at snapshot isolation as if the transaction started at the time of the WT_SESSION::snapshot call that created the snapshot.

通过使用包含snapshot = foo的配置调用WT_SESSION :: begin_transaction，可以从该快照“开始”启动后续事务。 该事务将以快照隔离方式运行，就好像该事务在创建快照的WT_SESSION :: snapshot调用时开始时一样。


Named snapshots keep data pinned in cache as if a real transaction were running for the time that the named snapshot is active. The resources associated with named snapshots should be released by calling WT_SESSION::snapshot with a configuration that includes "drop=". See WT_SESSION::snapshot documentation for details of the semantics supported by the drop configuration.

命名快照将数据固定在缓存中，就像在命名快照处于活动状态时正在运行实际事务一样。 与命名快照关联的资源应通过使用包含“ drop =”的配置调用WT_SESSION :: snapshot来释放。 有关放置配置支持的语义的详细信息，请参见WT_SESSION :: snapshot文档。

**Named snapshots are not durable: they do not survive WT_CONNECTION::close.**

#### Application-specified Transaction Timestamps

##### Timestamp overview

Some applications have their own notion of time, including an expected commit order for transactions that may be inconsistent with the order assigned by WiredTiger. We assume applications can represent their notion of a timestamp as an unsigned 64-bit integral value that generally increases over time. For example, a counter could be incremented to generate transaction timestamps, if that is sufficient for the application.

某些应用程序有自己的时间概念，包括可能与WiredTiger分配的顺序不一致的事务的预期提交顺序。 我们假设应用程序可以将其时间戳记表示为通常随时间增加的无符号64位整数值。 例如，如果对应用程序足够的话，可以增加一个计数器以生成事务时间戳。

Applications can assign explicit commit timestamps to transactions, then read "as of" a timestamp. The timestamp mechanism operates in parallel with WiredTiger's internal transaction ID management. It is recommended that once timestamps are in use for a particular table, all subsequent updates also use timestamps.

应用程序可以为事务分配明确的提交时间戳，然后“按原样”读取时间戳。 时间戳机制与WiredTiger的内部事务ID管理并行运行。 建议将时间戳用于特定表后，所有后续更新也都应使用时间戳。

##### Using transactions with timestamps

Applications that use timestamps will generally provide a timestamp at WT_SESSION::transaction_commit that will be assigned to all updates that are part of the transaction. WiredTiger also provides the ability to set a different commit timestamp for different updates in a single transaction. This can be done by calling WT_SESSION::timestamp_transaction repeatedly to set a new commit timestamp between a set of updates for the current transaction. This gives the ability to commit updates with different read "as of" timestamps in a single transaction.

使用时间戳的应用程序通常会在WT_SESSION :: transaction_commit时提供时间戳，该时间戳将分配给作为事务一部分的所有更新。 WiredTiger还提供了为单个事务中的不同更新设置不同的提交时间戳的功能。 这可以通过重复调用WT_SESSION :: timestamp_transaction来设置，以在当前事务的一组更新之间设置新的提交时间戳。 这样就可以在单个事务中使用不同的读取“截至”时间戳提交更新。

Setting a read timestamp in WT_SESSION::begin_transaction forces a transaction to run at snapshot isolation and ignore any commits with a newer timestamp.with different read "as of" timestamps in a single transaction.

在WT_SESSION :: begin_transaction中设置读取时间戳会强制事务以快照隔离运行，并忽略具有较新时间戳的任何提交。在单个事务中具有不同的“截至”时间戳。

Commit timestamps cannot be set in the past of any read timestamp that has been used. This is enforced by assertions in diagnostic builds, if applications violate this rule, data consistency can be violated.

不能在已使用的任何读取时间戳的过去设置提交时间戳。 这是由诊断版本中的断言强制执行的，如果应用程序违反此规则，则可能会破坏数据一致性。

The commits to a particular data item must be performed in timestamp order. If applications violate this rule, data consistency can be violated. Committing an update without a timestamp truncates the update's timestamp history and limits repeatable reads: no earlier version of the update will be returned regardless of the setting of the read timestamp.

对特定数据项的提交必须按时间戳顺序执行。 如果应用程序违反此规则，则可能会破坏数据一致性。 提交没有时间戳的更新会截断更新的时间戳历史记录，并限制可重复的读取：无论读取时间戳的设置如何，都不会返回更新的早期版本。

The WT_SESSION::prepare_transaction API is designed to be used in conjunction with timestamps and assigns a prepare timestamp to the transaction, which will be used for visibility checks until the transaction is committed or aborted. Once a transaction has been prepared the only other operations that can be completed are WT_SESSION::commit_transaction or WT_SESSION::rollback_transaction. The WT_SESSION::prepare_transaction API only guarantees that transactional conflicts will not cause the transaction to rollback - it does not guarantee that the transactions updates are durable. If a read operation encounters an update from a prepared transaction a WT_PREPARE_CONFLICT error will be returned indicating that it is not possible to choose a version of data to return until a prepared transaction is resolved, it is reasonable to retry such operations.

WT_SESSION :: prepare_transaction API旨在与时间戳结合使用，并为事务分配准备时间戳，该时间戳将用于可见性检查，直到事务被提交或中止。 一旦准备好交易，其他可以完成的操作就是WT_SESSION :: commit_transaction或WT_SESSION :: rollback_transaction。 WT_SESSION :: prepare_transaction API仅保证事务冲突不会导致事务回滚-不保证事务更新是持久的。 如果读取操作遇到了来自已准备好的事务的更新，将返回WT_PREPARE_CONFLICT错误，指示在解决已准备好的事务之前不可能选择要返回的数据版本，因此合理地重试此类操作。

Durability of the data updates performed by a prepared transaction, on tables configured with log=(enabled=false), can be controlled by specifying a durable timestamp during WT_SESSION::commit_transaction. Checkpoint will consider the durable timestamp, instead of commit timestamp for persisting the data updates. If the durable timestamp is not specified, then the commit timestamp will be considered as the durable timestamp.

可以通过在WT_SESSION :: commit_transaction期间指定持久时间戳来控制由准备好的事务在配置了log =（enabled = false）的表上执行的数据更新的持久性。 Checkpoint将考虑持久时间戳，而不是提交时间戳以持久保存数据更新。 如果未指定持久时间戳，则将提交时间戳视为持久时间戳。

为运行的事务分配时间戳有很多限制-下表总结了这些限制：




##### Automatic rounding of timestamps

##### Managing global timestamp state

##### Timestamp support in the extension API


### error handle

WiredTiger操作成功返回值0，错误返回非零值。 错误代码可以是正数也可以是负数：正错误代码是针对类似POSIX的系统所描述的标准错误代码（例如EINVAL或EBUSY），负错误代码是WiredTiger特定的（例如WT_ROLLBACK）。

WiredTiger特定的错误代码始终显示在-31,800至-31,999（含）范围内。

当对象不可用于独占访问时，WiredTiger对于需要独占访问的操作返回EBUSY。 例如，如果对象具有打开的游标，则WT_SESSION :: drop或WT_SESSION :: verify方法将失败。 请注意，内部WiredTiger线程可能会临时打开对象上的游标（例如，执行诸如统计日志记录之类的操作的线程），并且当对象上没有打开任何应用程序游标时，操作可能会暂时失败并返回EBUSY。

以下是WiredTiger特定返回值的完整列表：

- WT_ROLLBACK

This error is generated when an operation cannot be completed due to a conflict with concurrent operations. The operation may be retried; if a transaction is in progress, it should be rolled back and the operation retried in a new transaction.

当由于与并发操作冲突而导致操作无法完成时，将生成此错误。 该操作可以重试； 如果正在进行事务，则应回滚该事务，并在新事务中重试该操作。

- WT_DUPLICATE_KEY

This error is generated when the application attempts to insert a record with the same key as an existing record without the 'overwrite' configuration to WT_SESSION::open_cursor.

当应用程序尝试插入一条与现有记录具有相同键的记录，而没有对WT_SESSION :: open_cursor进行“覆盖”配置时，会产生此错误。

- WT_ERROR

This error is returned when an error is not covered by a specific error return.

当特定错误返回未涵盖错误时，将返回此错误。

- WT_NOTFOUND

This error indicates an operation did not find a value to return. This includes cursor search and other operations where no record matched the cursor's search key such as WT_CURSOR::update or WT_CURSOR::remove.

此错误表明操作未找到要返回的值。 这包括光标搜索和其他操作，其中没有记录与光标的搜索键匹配，例如WT_CURSOR :: update或WT_CURSOR :: remove。



- WT_PANIC

This error indicates an underlying problem that requires a database restart. The application may exit immediately, no further WiredTiger calls are required (and further calls will themselves immediately fail).

此错误表示需要重启数据库的潜在问题。 该应用程序可能会立即退出，不需要进一步的WiredTiger调用（并且进一步的调用本身将立即失败）。

- WT_RUN_RECOVERY

This error is generated when wiredtiger_open is configured to return an error if recovery is required to use the database.

如果将wiretiger_open配置为如果需要恢复才能使用该数据库，则将其配置为返回错误时，将生成此错误。

- WT_CACHE_FULL

This error is only generated when wiredtiger_open is configured to run in-memory, and an insert or update operation requires more than the configured cache size to complete. The operation may be retried; if a transaction is in progress, it should be rolled back and the operation retried in a new transaction.

仅在将wiretiger_open配置为在内存中运行并且插入或更新操作需要比配置的缓存大小更多的内存时，才会生成此错误。 该操作可以重试； 如果正在进行事务，则应回滚该事务，并在新事务中重试该操作。


- WT_PREPARE_CONFLICT

This error is generated when the application attempts to update an already updated record which is in prepared state. An updated record will be in prepared state, when the transaction that performed the update is in prepared state.

当应用程序尝试更新处于准备状态的已更新记录时，将生成此错误。 当执行更新的事务处于准备状态时，更新的记录将处于准备状态。


- WT_TRY_SALVAGE

This error is generated when corruption is detected in an on-disk file. During normal operations, this may occur in rare circumstances as a result of a system crash. The application may choose to salvage the file or retry wiredtiger_open with the 'salvage=true' configuration setting.

在磁盘文件中检测到损坏时，将生成此错误。 在正常操作期间，由于系统崩溃，在极少数情况下可能会发生这种情况。 应用程序可以选择抢救该文件或使用“ salvage = true”配置设置重试wiredtiger_open。

#### Translating errors

WT SESSION :: strerror和wiredtiger strerror函数返回与任何WiredTiger，ISO C或POSIX标准API关联的标准文本消息。

```cpp
const char *key = "non-existent key";
cursor->set_key(cursor, key);
if ((ret = cursor->remove(cursor)) != 0) {
fprintf(stderr, "cursor.remove: %s\n", cursor->session->strerror(cursor->session, ret));
return (ret);
}
```

Note that wiredtiger_strerror is not thread-safe.

```cpp

const char *key = "non-existent key";
cursor->set_key(cursor, key);
if ((ret = cursor->remove(cursor)) != 0) {
fprintf(stderr, "cursor.remove: %s\n", wiredtiger_strerror(ret));
return (ret);
}


```

#### Message handling using the WT_EVENT_HANDLER

可以通过将WT_EVENT_HANDLER的实现传递给wiredtiger_open或WT_CONNECTION :: open_session来配置特定的错误和其他消息处理。

例如，信息性消息和错误消息都可以传递到特定于应用程序的日志记录功能，该功能添加了时间戳并将该消息记录到文件中，并且错误消息还可以输出到stderr文件流。

Additionally, applications will normally handle WT_PANIC as a special case. WiredTiger will always call the error handler callback with WT_PANIC in the case of a fatal error requiring database restart, however, WiredTiger cannot guarantee applications will see an application thread return WT_PANIC from a WiredTiger API call. For this reason, a correctly-written WiredTiger application will likely specify at least an error handler which will immediately exit or otherwise handle fatal errors. Note that no further WiredTiger calls are required after an error handler is called with WT_PANIC (and further calls will themselves immediately fail).

此外，应用程序通常将处理WT_PANIC作为特殊情况。 如果发生致命错误，需要重新启动数据库，WiredTiger将始终使用WT_PANIC调用错误处理回调，但是，WiredTiger无法保证应用程序将看到应用程序线程从WiredTiger API调用返回WT_PANIC。 因此，正确编写的WiredTiger应用程序可能会至少指定一个错误处理程序，该程序将立即退出或处理致命错误。 请注意，在使用WT_PANIC调用错误处理程序之后，不需要其他WiredTiger调用（其他调用本身将立即失败）。


```cpp


/*
 * Create our own event handler structure to allow us to pass context through to event handler
 * callbacks. For this to work the WiredTiger event handler must appear first in our custom event
 * handler structure.
 */
typedef struct {
WT_EVENT_HANDLER h;
const char *app_id;
} CUSTOM_EVENT_HANDLER;
/*
 * handle_wiredtiger_error --
 *     Function to handle error callbacks from WiredTiger.
 */
int
handle_wiredtiger_error(
  WT_EVENT_HANDLER *handler, WT_SESSION *session, int error, const char *message)
{
CUSTOM_EVENT_HANDLER *custom_handler;
/* Cast the handler back to our custom handler. */
custom_handler = (CUSTOM_EVENT_HANDLER *)handler;
/* Report the error on the console. */
fprintf(stderr, "app_id %s, thread context %p, error %d, message %s\n", custom_handler->app_id,
  (void *)session, error, message);
/* Exit if the database has a fatal error. */
if (error == WT_PANIC)
exit(1);
return (0);
}
/*
 * handle_wiredtiger_message --
 *     Function to handle message callbacks from WiredTiger.
 */
int
handle_wiredtiger_message(WT_EVENT_HANDLER *handler, WT_SESSION *session, const char *message)
{
/* Cast the handler back to our custom handler. */
printf("app id %s, thread context %p, message %s\n", ((CUSTOM_EVENT_HANDLER *)handler)->app_id,
  (void *)session, message);
return (0);
}


CUSTOM_EVENT_HANDLER event_handler;
event_handler.h.handle_error = handle_wiredtiger_error;
event_handler.h.handle_message = handle_wiredtiger_message;
/* Set handlers to NULL to use the default handler. */
event_handler.h.handle_progress = NULL;
event_handler.h.handle_close = NULL;
event_handler.app_id = "example_event_handler";
error_check(wiredtiger_open(home, (WT_EVENT_HANDLER *)&event_handler, "create", &conn));

```
