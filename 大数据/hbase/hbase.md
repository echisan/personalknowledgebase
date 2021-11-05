- 什么时候需要用hbase
  - 非常大量的数据需要存储
  - 需求不会用到RDBMS额外的特性，比如行类型(typed columns)，二级索引、事务、高级的sql功能等
  - 拥有足够的硬件资源

# 数据模型

- Table

  由很多行(rows)组成

- Row

  由一个row Key跟一个或多个列关联的值；

  行按照row Key字母排序；

  rowKey的设计很重要；目标是将相关的行存储在附近按照rowkey的字母排序；

- Column

  由column family跟column限定词组成，由`:`分割，例如`persion:name`

- Column Family

  在物理意义上的列跟值；也就是一个ColumnFamily一组物理文件；主要是为了性能；

  每个columnFamily有一组存储属性，比如此columnfamily的值是否应该缓存到内存、数据怎么压缩又或者rowkey被编码过的等等；

  表里每行都有相同的列族，虽然给定的列在该列组里没有存储；

- Column Qualifier

  列的限定词；

  虽然列组在创建的时候就不会再变了，但是列可以被修改，所以每行可能有很大的区别；

- Cell

  单元格；一个单元格包含行、列祖、列限定词、值、时间戳

- Timestamp

  每个value都包含一个时间戳；是一个值value的版本的标识；

  默认情况下，表示当数据被写入时RegionServer的时间戳；不过在写入数据到单元格的时候可以指定不一样的时间戳；

## 概念上的视图

用表来表示的话

[webtable](https://www.notion.so/d86a215c06934f65b1912e7613f69f2c)

用json表示的话

```json
{
  "com.cnn.www": {
    contents: {
      t6: contents:html: "<html>..."
      t5: contents:html: "<html>..."
      t3: contents:html: "<html>..."
    }
    anchor: {
      t9: anchor:cnnsi.com = "CNN"
      t8: anchor:my.look.ca = "CNN.com"
    }
    people: {}
  }
  "com.example.www": {
    contents: {
      t5: contents:html: "<html>..."
    }
    anchor: {}
    people: {
      t5: people:author: "John Doe"
    }
  }
}
```

## 物理视图

虽然在概念视图上看起来一行是个稀疏的集合，但是他们被物理地存储在列祖中；一个新的列限定符可以随时被添加到一个已经存在的列祖中；

[ColumnFamily anchor](https://www.notion.so/3b16118e19ed477abed899b1c66cee14)

[ColumnFamily contents](https://www.notion.so/d1d3257018b2442b8ada19bf92711212)

在概念视图中空的单元格其实是完全没有存储的；所以请求contents:html 在t8时间戳的列会返回no vlaue；

如果没有提供时间戳，则会返回一个最近的值；

就算提供了多个版本，最近的值还是会第一个被找到，因此时间戳是以一个降序的顺序；

## Namespace

类似于RDBMS的database，这是实现多租户的基础：

- 磁盘管理：限定一个命名空间下可以使用多少的资源；https://issues.apache.org/jira/browse/HBASE-8410
- 命名空间安全管理：提供对租户的另一层次的安全管理；https://issues.apache.org/jira/browse/HBASE-9206
- RegionServer分组：A namespace/table can be pinned onto a subset of RegionServers thus guaranteeing a coarse level of isolation.https://issues.apache.org/jira/browse/HBASE-6721

### 命名空间管理

命名空间可以被创建、删除、修改；

命名空间的成员在创建表的时候以全限定性名决定；比如：`<table namespace>:<table qualifier>`

示例：

```bash
# create a namespace
create_namespace 'my_ns'
# create my_table in my_ns namespace
create 'my_ns:my_table', 'fam'
# drop namespace
drop_namespace 'my_ns'
# alter namespace
alter_namespace 'my_ns' , {METHOD => 'set', 'PROPERTY_NAME' => 'PROPERTY_VALUE'}
```

### 预定义命名空间

有两个预定义的命名空间

- hbase - 系统明明空间，用来包含hbase一些内部的表
- default - 当没特别指定命名空间的情况下，会自动使用这个命名空间

```bash
# namespace=foo and table qualifier=bar
create 'foo:bar', 'fam'
# namespace=default and table qualifier=bar
create 'bar', 'fam'
```

## Table

Tables are declared up front at schema definition time.

## Row

Row keys是一个未解释的字节数组；

行是排序的，以最小的在表的最前面这种顺序；

## Column Family

在hbase里，列Column是被归到列祖里的；

所有列祖的成员都有同样的前缀；

列祖的前缀必须由可打印的字符组成；

列限定符可以由任意的字符组成；

定义schema的时候列祖一定要先定义好；

物理上，列族成员都是存在一块的；

## Cells

A {row, column, version} tuple exactly specifies a cell in HBase. Cell content is uninterpreted bytes

## 数据模型的操作

有4个主要的操作，get, put, scan, delete；这些都是对一个表示例进行操作的；

### Get

获取指定行的一些属性；`Table.get`

### Put

如果行不存在的话就新增一行，如果存在则会更新行；`Table.put`,`Table.batch`

### Scans

可以根据指定的属性，扫描多行

用法如下：

假设rowkey为 `row1,row2,row3` 其他一些rowkey为 `abc1,abc2,abc3`

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...

Table table = ...      // instantiate a Table instance

Scan scan = new Scan();
scan.addColumn(CF, ATTR);
scan.setRowPrefixFilter(Bytes.toBytes("row"));
ResultScanner rs = table.getScanner(scan);
try {
  for (Result r = rs.next(); r != null; r = rs.next()) {
    // process result...
  }
} finally {
  rs.close();  // always close the ResultScanner!
}
```

如果要停止扫描的话，最简单的方式就是使用`InclusiveStopFilter`

### Delete

删除一行， `Table.delete`

hbase不会在对应的地方修改数据，删除操作会创建一个新的标志，叫墓碑；

这些墓碑标志以及那些被删的行的数据都会被压缩；

## Versions

`{row, column, version}` 可以指定一个cell；尽管row跟column用字节数组表示，但是version是以long类型表示；

version是以降序保存，所以最新的值会最先被发现；

一些疑惑的点：

- 如果写入多个相同version的cell， 那么最后一个写入的才可以被获取到
- 可以写入非升序的version

### 指定版本号数

版本号数的最大值在创建列的时候可以指定或者是在创建table的时候指定，也可以通过 `alter` 命令，或者 `HColumnDescriptor.DEFAULT_VERSIONS`

在HBase0.96版本之前version默认值是`3`，在之后默认值变为`1`

为列祖修改version最大值

```bash
hbase> alter 't1', NAME => 'f1' , VERSION => 5
```

修改最小值

```bash
hbase> alter 't1', NAME => 'f1' , MIN_VERSION => 2
```

从Hbase0.98.2开始，可以全局配置最大版本数，在 `hbase-site.xml` 中配置 `hbase.column.max.version`

### Versions and HBase操作

1. Get/Scan

Scan实现了Get接口

默认情况下，在没特别指定版本的时候，会返回一个version最大的cell（不一定是最新写入的）；

可以使用如下修改默认的情况

1. 如果需要获取多个版本的, `Get.setMaxVersions()`

2. 按照时间返回version, `Get.setTimeRange()`

   如果想要指定某个时间的version,但又不是最新的，可以range 0 到你想要的版本时间，然后再setMaxVersions(1)

3. 默认Get行为的实例

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Get get = new Get(Bytes.toBytes("row1"));
Result r = table.get(get);
byte[] b = r.getValue(CF, ATTR);  // returns current version of value
```

1. Version Get Example

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Get get = new Get(Bytes.toBytes("row1"));
get.setMaxVersions(3);  // will return last 3 versions of row
Result r = table.get(get);
byte[] b = r.getValue(CF, ATTR);  // returns current version of value
List<Cell> cells = r.getColumnCells(CF, ATTR);  // returns all versions of this column
```

1. Put

put操作永远会创建新版本；

默认情况下系统会使用`currentTimeMillis` 作为版本，不过可以指定一个版本（long integer)，咦每一列为维度；

如果要覆盖已存在的值，可以指定相同的行，列，version

隐式版本实例，version为时间戳

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Put put = new Put(Bytes.toBytes(row));
put.add(CF, ATTR, Bytes.toBytes( data));
table.put(put);
```

显式指定版本号

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...
Put put = new Put( Bytes.toBytes(row));
long explicitTimeInMs = 555;  // just an example
put.add(CF, ATTR, explicitTimeInMs, Bytes.toBytes(data));
table.put(put);
```

> 警告：hbase内部使用时间戳作为版本号用于计算time-to-live(ttl)；最好不好自己把时间戳set进去；推荐可以把时间戳作为单独的一列，又或者可以把时间戳作为rowkey的一部分；

Cell Version Example

```java
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...

Put put = new Put(Bytes.toBytes(row));
put.add(put.getCellBuilder().setQualifier(ATTR)
   .setFamily(CF)
   .setValue(Bytes.toBytes(data))
   .build());
table.put(put);
```

1. Delete

有三种不同类型的删除；

1. Delete: 删除指定版本的列
2. Delete column: 删除所有版本的列
3. Delete family: 删除列祖下所有的列

当删除行的时候，hbase内部会为每个列祖创建一个tombstone（不是为每个列创建）

删除操作不会将数据直接删掉或者将数据标识为删除，也不会在对应的位置上修改数据，而是会创建一个tombstone作为标志，这个tombstone会记录哪些值是需要被删除的；在hbase执行major compaction的时候，这些数据以及tombstone都会被删除。

如果你指定一个比任意已给版本号都要大的版本去删除，那么可以考虑直接将整行删除；

删除的标志会在下次major compaction的时候清理掉，除非在列祖上配置该参数 `KEEP_DELETED_CELLS`

想要设置删除的Cell保存多久可以通过在`hbase-site.xml`中配置`bhase.hstore.time.to.purge.deletes`设置delete TTL

**在Hbase2.0.0后可选的新版本以及删除行为**

hbase-2.0.0后，operator可以指定交换的版本；可以通过列属性 `NEW_VERSION_BEHAVIOR` 为`true`修改删除行为，如果要设置列族，需要先把table disable掉

The 'new version behavior', undoes the limitations listed below whereby a `Delete` ALWAYS overshadows a `Put` if at the same location — i.e. same row, column family, qualifier and timestamp — regardless of which arrived first. Version accounting is also changed as deleted versions are considered toward total version count. This is done to ensure results are not changed should a major compaction intercede. See `HBASE-15968` and linked issues for discussion.

Running with this new configuration currently costs; we factor the Cell MVCC on every compare so we burn more CPU. The slow down will depend. In testing we’ve seen between 0% and 25% degradation.

**Deletes mask Puts**

就算是puts操作是在delete之后也有可能delete操作会覆盖掉put操作；

要记住一点的是，删除操作时创建一个tombstone，只有在经过major compaction操作之后数据才会消失；

假如你删除了所有东西（delete of everything）≤ T这个时间戳为条件，然后你再往里面插入一个时间戳≤T的数据，就算你是在删除操作之后进行put操作的，还是有可能被tombstone操作覆盖掉；

虽然put操作没有报错，但是在你进行get操作之后，你可能会发现你put操作没有生效，当然在major compaction操作之前还是正常工作的；

如果你自增的数作为版本号的话，这不会产生这个问题；

如果你不在意这个时间的话，还是有可能发生这样的问题，比如你以很快的速度执行了delete和put操作，如果这两个操作的时间(millisecond)刚好相同的话；

**Major compactions change query results**

假设你创建了三个cell在时间t1, t2, t3， 然后最大版本设置为2；

这时候你获取所有版本， 会返回t2, t3

但是你删除了t2或者t3的版本，t1会重新出现；

一旦major compaction执行了，上面的情况就不会再出现了；

## Sort Order

All data model operations HBase return data in sorted order. First by row, then by ColumnFamily, followed by column qualifier, and finally timestamp (sorted in reverse, so newest records are returned first).

## Column Metadata

只有列祖内部的keyValue实例保存了列的元数据，因此hbase每行不仅可以支持大量的列以及各种类型的列；

The only way to get a complete set of columns that exist for a ColumnFamily is to process all the rows.

## Joins

hbase不支持这玩意，hbase的读操作只有get跟scan；

可以自己程序去做join操作

## ACID

https://hbase.apache.org/acid-semantics.html

http://hadoop-hbase.blogspot.com/2012/03/acid-in-hbase.html

# HBase and Schema Design

A good introduction on the strength and weaknesses modelling on the various non-rdbms datastores is to be found in Ian Varley’s Master thesis, [No Relation: The Mixed Blessings of Non-Relational Databases](http://ianvarley.com/UT/MR/Varley_MastersReport_Full_2009-08-07.pdf). It is a little dated now but a good background read if you have a moment on how HBase schema modeling differs from how it is done in an RDBMS. Also, read [keyvalue](https://hbase.apache.org/book.html#keyvalue) for how HBase stores data internally, and the section on [schema.casestudies](https://hbase.apache.org/book.html#schema.casestudies).

bigtable schema设计

https://cloud.google.com/bigtable/docs/schema-design

## Schema Creation

可以通过Hbase shell或者Admin java API去操作

当修改列族配置的时候，需要先将表disabled掉

```java
Configuration config = HBaseConfiguration.create();
Admin admin = new Admin(conf);
TableName table = TableName.valueOf("myTable");

admin.disableTable(table);

HColumnDescriptor cf1 = ...;
admin.addColumn(table, cf1);      // adding new ColumnFamily
HColumnDescriptor cf2 = ...;
admin.modifyColumn(table, cf2);    // modifying existing ColumnFamily

admin.enableTable(table);
```

### Schema Updates

当对tables或者columnfamilies做了修改（比如：region size, block size），这些修改都会在下一次major compaction执行完之后生效；此时storeFiles会被重写；

## Table Schema Rules Of Thumb

一些仅供参考的小建议

- Aim to have regions sized between 10 and 50 GB.
- Aim to have cells no larger than 10 MB, or 50 MB if you use [mob](https://hbase.apache.org/book.html#hbase_mob). Otherwise, consider storing your cell data in HDFS and store a pointer to the data in HBase.
- 一个标准的schema，表包含的列祖应该在1~3个；设计hbase的表不能模仿RDBMS的表设计；
- Around 50-100 regions is a good number for a table with 1 or 2 column families. Remember that a region is a contiguous segment of a column family.（region是列祖的切片）
- 列族的名称越短越好；The column family names are stored for every value (ignoring prefix encoding). They should not be self-documenting and descriptive like in a typical RDBMS.
- 如果存时序数据的话，一般是deviceId或者serviceId加上时间戳，you can end up with a pattern where older data regions never have additional writes beyond a certain [age.In](http://age.In) this type of situation, you end up with a small number of active regions and a large number of older regions which have no new writes. For these situations, you can tolerate a larger number of regions because your resource consumption is driven by the active regions only.（也就是说这种deviceID+TS的方式，可以做到旧的region基本不会写数据）
- If only one column family is busy with writes, only that column family accomulates memory. Be aware of write patterns when allocating resources.

## RegionServer Sizing Rules of Thumb

http://hadoop-hbase.blogspot.com/2013/01/hbase-region-server-memory-sizing.html

结论就是，你需要的内存比你想象的要多。

> Personally I would place the maximum disk space per machine that can be served exclusively with HBase around 6T, unless you have a very read-heavy workload. In that case the Java heap should be 32GB (20G regions, 128M memstores, the rest defaults).

## On the number of column families

现在hbase还不能很好地支持超过2~3个列祖，所以需要控制好列祖地数量。

现在刷新操作是以Region为基础完成的，所以一个列祖需要刷新，会顺带将相邻的列祖一块刷新，如果有很多个列祖的话，会带来一些额外的刷新操作；

### Cardinality of ColumnFamilies(列族的行数)

如果一个表中有多个列族，需要在意两个列族的行数；

比如列族A有100万行，而列族B有一亿行，

## Rowkey Design

### Hotspotting(局部热点？)

Hbase是以字典序存储数据的，这个设计是为了scan最佳化，允许将相关的行存储在一起；然而辣鸡设计会导致rowkey存在局部热点；局部热点会导致大量的流量直接访问同一个节点或者少量几个节点或者同个集群；这些流量可以是读、写或者其他操作；

一些避免热点问题的技巧

**Salting（加盐）**

在这个场景中，纯加盐，不带任何加密算法。单纯在rowkey上加个随机前缀；

这种方式优势是可以增加写入得吞吐量，但是会导致读得耗时增加；

例如

```bash
# rowkey如下
foo0001
foo0002
foo0003
foo0004
# 加盐，加前缀，比如用abcd作为前缀
a-foo0003
b-foo0001
c-foo0004
d-foo0002
# 
a-foo0003
b-foo0001
c-foo0003
c-foo0004
d-foo0002
```

**Hashing(看不懂啊)**

可以使用单向hash替代随机的assignment；散列，这种方式可以根据rowkey确定hash，反推出已存在文件中的rowkey是什么，从而直接找到该行

示例：

Given the same situation in the salting example above, you could instead apply a one-way hash that would cause the row with key foo0003 to always, and predictably, receive the a prefix. Then, to retrieve that row, you would already know the key. You could also optimize things so that certain pairs of keys were always in the same region, for instance.

[HBase之rowkey设计原则和方法_zhangvalue的博客-CSDN博客_hbase的rowkey设计原则](https://blog.csdn.net/zhangvalue/article/details/100636697)

**时间戳反转Reversiong the Key**

避免数据热点，因为时间戳的开头都是一致的，导致会一直在已给region后追加，导致数据热点；

反转时间戳后，因为秒数经常变化的，导致rowkey的顺序发生了变化，从而追加到的region就分散了；

## Monotonically Increasing Row Keys/Timeseries Data单调递增的时序数据

不要直接用时间戳作为rowkey， 而是应该由[metric_type]+[event_timestamp]， metric可能由若干个到几百个；

[App Engine datastore tip: monotonically increasing values are bad - Ikai Says...](https://ikaisays.com/2011/01/25/app-engine-datastore-tip-monotonically-increasing-values-are-bad/)

## Try to minimize row and column sizes尝试减少行和列的sizes

在hbase，值（values）总是和坐标被一起运送的；随着cell的值在系统中传输，行，列名，时间戳（row, column, timestamp)会一起带上。

如果你的行跟列的名字很长，特别是跟值比较的话（比如说value就一个int类型的值，才占4个字节，但是列名或者行名就占16个字节了，）会出现一些奇怪的问题：

https://issues.apache.org/jira/browse/HBASE-3551?page=com.atlassian.jira.plugin.system.issuetabpanels%3Acomment-tabpanel&focusedCommentId=13005272#comment-13005272

## Constraints（约束限制）

索引信息是保存在hbase的storefiles里（StoreFile(HFile)），为了提升随机访问的速度，又因为cell的坐标信息很大，需要开辟一大块内存空间，导致oom；

Mark in the above cited comment suggests upping the block size so entries in the store file index happen at a larger interval or modify the table schema so it makes for smaller rows and column names. Compression will also make for larger indices.

### **Column Families**

名称越短越好，最好一个字符；比如拿`d`去表示`data/default`

### **Attributes**

虽然详细的属性名很易读，但是还是小一点比较好

### **RowKey Length**

rowkey也是短一点好，不过也要设计的方便去访问；

rowkey再短，也没有一个长一点的key但是访问性能好；

### **Byte Patterns**

就是最好用int就int，不用string类型去存数字，因为占用的空间是不一样的

If you stored this number as a String — presuming a byte per character — you need nearly 3x the bytes.

## Reverse Timestamps

在hbase0.98版本后，支持扫描表顺序反过来，用法是 `Scan.setReversed()`

一些经典案例，怎么实现rowkey以及结构的设计；

数据库处理中的一个常见问题是快速找到值的最新版本；使用反转时间戳可以很好的解决这样的问题；

用法如： `reverse_timestamp = (Long.MAX_VALUE - timestamp)`  `[key][reverse_timestamp]`

因为hbase是字典序，新的行会放在最前面；

## Rowkeys and ColumnFamilies

Rowkeys的范围是columnfamilies，所以，同样的rowkey可以在同一个table内，存在于不同的列族中而没有冲突

## Immutability of Rowkeys (rowkeys的不可变性)

rowkeys没法改变，唯一“改变”的方式只有删除行或者重新插入数据到同样的行；

## Relationship Between RowKeys and Region Splits

rowkey跟region切割的关系；

1. pre-splitting表通常是最佳实践，需要以某种方式去预拆分它们，即键空间中的所有区域都可以访问。虽然此示例演示了十六进制键键空间的问题，但任何键空间都可能发生同样的问题。
2. While generally not advisable, using hex-keys (and more generally, displayable data) can still work with pre-split tables as long as all the created regions are accessible in the keyspace.

# Number of Versions

## 版本的最大值

还有另外一种经常在dist-list中被提及的模式，叫"bucketing"时间戳，通过对时间戳 取余，如果面向时间的扫描（time-oriented scans）很重要的话，那么这会很有用；要注意就是自己定义个桶数了；

as stated above如上所述，去选择特定时间区间，需要为每个桶执行扫描。

默认min_version为0，表示这个特性没有开启；

可以用ttl去实现

# Supported Datatypes

hbase支持字节数组进去字节数组出来的特性；

能存各种类型，只要能序列化成字节数组跟反序列化回来；

### Counters

One supported datatype that deserves special mention are "counters" (i.e., the ability to do atomic increments of numbers). See [Increment](https://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#increment(org.apache.hadoop.hbase.client.Increment)) in `Table`.

Synchronization on counters are done on the RegionServer, not in the client.

# Joins

不支持，要么设计好表，要么程序里干

# Time To Live（TTL）

列族可以配置TTL（单位为秒），hbase会时间到期时自动删除这些行（包括所有version），TTL被hbase解析成UTC时间戳。

仅包含过期行的存储文件在次要压缩时被删除。

例如有100个桶，为了找出一条特定时间的数据，要扫描100个桶去获得数据，所以需要权衡。

1. **Host In The Rowkey Lead Position（Host 在rowkey的开头）**

The rowkey `[hostname][log-event][timestamp]` is a candidate if there is a large-ish number of hosts to spread the writes and reads across the keyspace. This approach would be useful if scanning by hostname was a priority.

1. **Timestamp, or Reverse Timestamp?**

如果有需求是获取最近的数据，那么可以反转时间戳， `timestamp=Long.MAX_VALUE - timestamp`

1. **Variable Length or Fixed Length Rowkeys?(可变长度还是固定长度的rowkeys)**

要注意一点就是如果hostname是`a`，eventType是`e1`，那么rowkey很短，没问题；