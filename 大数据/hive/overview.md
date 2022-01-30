# overview

`hive.metastore.warehouse.dir` 配置数据库所在的目录

通过 `create database $DB_NAME LOCATION '/my/perferred/directory'` 修改默认位置

- 内部表（由hive管理生命周期）
- 外部表 

- - `CREATE EXTERNAL TABLE ...`

如果表中数据以及分区个数都非常大，执行这样一个包含所有分区的查询可能会出发一个巨大的mapreduce任务。建议将Hive设置为”strict严格”模式，对于where字句没有加分区过滤的话，会禁止提交这个任务。

```
set hive.mapred.mode=strict
```

`ALTER TABLE` 语句可以单独进行增加分区。

例如：(按照分层目录结构组织)

```bash
ALTER TABLE log_message ADD PARTITION(year = 2012, month = 1, day = 2)
LOCATION 'hdfs://master_server/data/log_messages/2012/01/02';
```

利用该灵活性，可以使用Amazon S3这样廉价的设备存储旧的数据

```bash
# 1. 将分区下的数据拷贝到S3中， 可以使用hadoop distcp
hadoop distcp /data/log/log_message/2011/12/02 s3n://ourbucket/logs/2011/12/02
# 2. 修改表，将分区路径指向到S3路径
ALTER TABLE log_messages PARTITION(year = 2011, month = 12, day = 2) 
SET LOCATION 's3n://ourbucket/logs/2011/01/02';
# 3. 使用hadoop fs -rmr 命令删除hdfs中的这个分区数据
hadoop fs -rmr /data/log_messages/2011/01/02
```

## 删除或替换列

`REPLACE`语句只能用于使用了如下两种SerDe模块的表

- DynamicSerDe
- MetadataTypedColumnsetSerDe

### 表属性

可以增加附加的表属性或者修改已存在的属性，但无法删除属性

```bash
ALTER TABLE log_message SET TBLPROPERTIES(
 'notes' = 'the process id is no longer captured; this column is always NULL');
```

## HQL



## hiveserver2

配置hiveserver2

1. 配置hadoop core-site.xml允许访问
2. 配置hive hive-site.xml账号密码以及hiveserver2端口

core-site.xml

```xml
<property>
        <name>hadoop.proxyuser.echisan.hosts</name>
        <value>*</value>
</property>
<property>
        <name>hadoop.proxyuser.echisan.groups</name>
        <value>*</value>
</property>
```

hive-site.xml

```xml
<property>
    <name>hive.server2.thrift.port</name>
    <value>10000</value>
</property>
<property>
  <name>beeline.hs2.connection.user</name>
  <value>echisan</value>
</property>
<property>
  <name>beeline.hs2.connection.password</name>
  <value>echisan</value>
</property>
```

## DML

表中装载数据

`load data [local] inpath '${data_path}' [overwrite] into table $table [partition(partcol=val1,...)]; `

*overwrite: 表示覆盖表中已有数据，否则表示追加

### 通过查询语句向表中插入数据（Insert）

#### 多表多分区插入模式（根据多张表查询结果）

```sql
from student
insert overwrite table student partition (month = '201707')
select id,name where month = '201707'
insert overwrite table student partition (month = '201706')
select id,name where month = '201706';
```

### 查询语句中创建表并加载数据（As Select）

```sql
create table if not exists student3
as
select id, name
from student;
```

### 创建表时通过 Location 指定加载数据路径

```sql
create external table if not exists student5(
    id int, name string
)
row format delimited fields terminated by '\t'
location '/student';
```

### 修改表location
```sql
alter table lag_test set location '/user/echisan/hive/data/lag_test_data.txt';
```

###  Import 数据到指定 Hive 表中

```sql
import table student2 from '/user/echisan/hive/data/export/student';
```

### 数据导出

#### insert导出

```sql
# 1) 将查询的结果导出到本地
insert overwrite local directory '/home/echisan/bigdata/test-data/insert_export/student' select * from student;
# 2) 将查询的结果格式化导出到本地
insert overwrite local directory '/home/echisan/bigdata/test-data/insert_export/student'
row format delimited fields terminated by '\t'
select * from student;
# 3) 将查询的结果导出到 HDFS 上(没有 local)
```

### Sqoop导出

// todo

## 查询

### JOIN

优化：当对 3 个或者更多表进行 join 连接时，如果每个 on 子句都使用相同的连接键的 话，那么只会产生一个 MapReduce job。

### 分区(Distribute By)

Distribute By： 在有些情况下，我们需要控制某个特定行应该到哪个 reducer，通常是为 了进行后续的聚集操作。distribute by 子句可以做这件事。distribute by 类似 MR 中 partition （自定义分区），进行分区，结合 sort by 使用

例子：先按照部门编号分区，再按照员工编号降序排序。

> distribute by 的分区规则是根据分区字段的 hash 码与 reduce 的个数进行模除后， 余数相同的分到一个区。
>
> Hive 要求 DISTRIBUTE BY 语句要写在 SORT BY 语句之前。

### Cluster By

当 distribute by 和 sorts by 字段相同时，可以使用 cluster by 方式。 cluster by 除了具有 distribute by 的功能外还兼具 sort by 的功能。但是排序只能是升序 排序，不能指定排序规则为 ASC 或者 DESC。

```sql
# 以下两种写法等价
select * from emp cluster by deptno;
select * from emp distribute by deptno sort by deptno asc ;
```

## 分区表和分桶表

### 分区表

1. 正常加载数据

```sql
load data local inpath 'dept_20200403.log' into table dept_partition partition(day='20200403');
```

2. 把数据直接上传到分区目录上，让分区表和数据产生关联的三种方式

1） 方式一：上传数据后修复

```shell
echisan@ubuntu:~/bigdata/test-data/dept_partition_test_data$ vim dep_20200404.log
echisan@ubuntu:~/bigdata/test-data/dept_partition_test_data$ hdfs dfs -mkdir /user/hive/warehouse/dept_partition/day=20200404
echisan@ubuntu:~/bigdata/test-data/dept_partition_test_data$ hdfs dfs -put dep_20200404.log /user/hive/warehouse/dept_partition/day=20200404
echisan@ubuntu:~/bigdata/test-data/dept_partition_test_data$ hive -e "msck repair table dept_partition;"
```

2）方式二：上传数据后添加分区

```shell
1. 上传数据
2. 执行添加分区
3. 查询数据
```

3）创建文件夹后 load 数据到分区

1. 创建目录
2. 上传数据
3. 查询数据

#### 动态分区调整

关系型数据库中，对分区表 Insert 数据时候，数据库自动会根据分区字段的值，将数据 插入到相应的分区中，Hive 中也提供了类似的机制，即动态分区(Dynamic Partition)，只不过， 使用 Hive 的动态分区，需要进行相应的配置。

1. 开启动态分区参数设置

1）开启动态分区功能（默认 true，开启）

`hive.exec.dynamic.partition=true`

2 ) 设置为非严格模式（动态分区的模式，默认 strict，表示必须指定至少一个分区为 静态分区，nonstrict 模式表示允许所有的分区字段都可以使用动态分区。）

`hive.exec.dynamic.partition.mode=nonstrict`

3 ) 在所有执行MR的节点上,最大一共可以创建多少个动态分区.默认1000

`hive.exec.max.dynamic.partitions=1000`

4 )   在每个执行 MR 的节点上，最大可以创建多少个动态分区。该参数需要根据实际 的数据来设定。

`hive.exec.max.dynamic.partitions.pernode=100`

5 ) 整个 MR Job 中，最大可以创建多少个 HDFS 文件。默认 100000

`hive.exec.max.created.files=100000`

6 ) 当有空分区生成时，是否抛出异常。一般不需要设置。默认 false

`hive.error.on.empty.partition=false`

### 分区桶

分区提供一个隔离数据和优化查询的便利方式。不过，并非所有的数据集都可形成合理 的分区。对于一张表或者分区，Hive 可以进一步组织成桶，也就是更为细粒度的数据范围 划分。

 分桶是将数据集分解成更容易管理的若干部分的另一个技术。 

**分区针对的是数据的存储路径；分桶针对的是数据文件。**

#### 分桶表操作需要注意的事项

1. reduce 的个数设置为-1,让 Job 自行决定需要用多少个 reduce或者将 reduce 的个数设置为大于等于分桶表的桶数
2. 从 hdfs 中 load 数据到分桶表中，避免本地文件找不到问题
3. 不要使用本地模式

## 抽样查询

对于非常大的数据集，有时用户需要使用的是一个具有代表性的查询结果而不是全部结 果。Hive 可以通过对表进行抽样来满足这个需求。

// todo 不是很懂

```sql
select * from stu_buck tablesample ( bucket 1 out of 4);
```

# 函数

## 常用内置函数

1. NVL：给值为 NULL 的数据赋值，它的格式是 NVL( value，default_value)。
2. CASE WITH THEN ELSE END

### 行转列

- CONCAT(string A/col, string B/col…)：返回输入字符串连接后的结果，支持任意个输入字 符串; 
- CONCAT_WS(separator, str1, str2,...)：它是一个特殊形式的 CONCAT()。第一个参数剩余参 数间的分隔符。分隔符可以是与剩余参数一样的字符串。如果分隔符是 NULL，返回值也将 为 NULL。这个函数会跳过分隔符参数后的任何 NULL 和空字符串。分隔符将被加到被连接 的字符串之间;
- collect_set

```sql
select concat_ws(',', p.constellation, p.blood_type),
       concat_ws('|', collect_set(p.name))
from person_info p
group by p.constellation,p.blood_type;
```

### 列转行

- EXPLODE(col): 将 hive 一列中复杂的 Array 或者 Map 结构拆分成多行。

- LATERAL VIEW

用法：`LATERAL VIEW udtf(expression) tableAlias AS columnAlias`

用于和 split, explode 等 UDTF 一起使用，它能够将一列数据拆成多行数据，在此 基础上可以对拆分后的数据进行聚合。

```sql
select movie,
       category_name
from movie_info
lateral view explode(split(category, ",")) movie_info_tmp as category_name;
```

### 窗口函数（开窗函数）

- OVER()：指定分析函数工作的数据窗口大小，这个数据窗口大小可能会随着行的变而变化
  - CURRENT ROW：当前行
  - n PRECEDING：往前 n 行数据
  - n FOLLOWING：往后 n 行数据
  - UNBOUNDED：起点，
    - UNBOUNDED PRECEDING 表示从前面的起点，
    - UNBOUNDED FOLLOWING 表示到后面的终点
  - LAG(col,n,default_val)：往前第 n 行数据
  - LEAD(col,n, default_val)：往后第 n 行数据
  - NTILE(n)：把有序窗口的行分发到指定数据的组中，各个组有编号，编号从 1 开始，对 于每一行，NTILE 返回此行所属的组的编号。
  - **注意：n 必须为 int 类型。**

```sql
select name, orderdate, cost,
# 所有行相加
sum(cost) over () as sample1
# 按name分组，组内数据相加
sum(cost) over (partition by name) as sample2
# 按name分组，根据orderdate排序，组内数据累加
sum(cost) over (partition by name order by orderdate) as sample3
# 与sample3一样，由起点到当前行的聚合
sum(cost) over (partition by name order by orderdate rows between unbounded preceding and current row ) as sample4
# 当前行与前面一行做聚合
sum(cost) over (partition by name order by orderdate rows between 1 preceding and current row ) as sample5
# 当前行和前面一行及后面一行
sum(cost) over (partition by name order by orderdate rows between 1 preceding and 1 following) as sample6
# 当前行及后面所有行
sum(cost) over (partition by name order by orderdate rows between current row and unbounded following) as sample7
from business;
```

rows 必须跟在 order by 子句之后，对排序的结果进行限制，使用固定的行数来限制分 区中的数据行数量

- LAG  (scalar_expression [,offset] [,default]) OVER ([query_partition_clause] order_by_clause); The LAG function is used to access data from a previous row.

- LEAD(col,n,DEFAULT)与LAG相反, 用于统计窗口内往下第n行值

- NTILE(n)：用于将分组数据按照顺序切分成n片，返回当前记录所在的切片值
  - 参考：https://www.cnblogs.com/skyEva/p/7552129.html
  
- first_value(),last_value(): 
  - 获取分组最后一个值`first_value(url) over (partition by cookieid order by createtime desc ) last2`
  
- RANK

  - RANK() 排序相同时会重复，总数不会变

  - DENSE_RANK() 排序相同时会重复，总数会减少

  - ROW_NUMBER 会根据顺序计算

  - ```sql
    select name,
           subject,
           score,
           rank() over (partition by subject order by score desc) rp,
           dense_rank() over (partition by subject order by score desc ) drp,
           row_number() over (partition by subject order by score desc ) rmp
    from score;
    +----+-------+-----+--+---+---+
    |name|subject|score|rp|drp|rmp|
    +----+-------+-----+--+---+---+
    |孙悟空 |数学     |95   |1 |1  |1  |
    |宋宋  |数学     |86   |2 |2  |2  |
    |婷婷  |数学     |85   |3 |3  |3  |
    |大海  |数学     |56   |4 |4  |4  |
    |宋宋  |英语     |84   |1 |1  |1  |
    |大海  |英语     |84   |1 |1  |2  |
    |婷婷  |英语     |78   |3 |2  |3  |
    |孙悟空 |英语     |68   |4 |3  |4  |
    |大海  |语文     |94   |1 |1  |1  |
    |孙悟空 |语文     |87   |2 |2  |2  |
    |婷婷  |语文     |65   |3 |3  |3  |
    |宋宋  |语文     |64   |4 |4  |4  |
    +----+-------+-----+--+---+---+
    ```



## 自定义函数

- 根据用户自定义函数类别分为以下三种

  1. UDF（User-Defined-Function）

     一进一出

  2. UDAF（User-Defined Aggregation Function）

     聚合函数，多进一出，类似于:count/max/min

  3. UDTF(User-Defined Table-Generating Functions)

     一进多出，如：lateral view explode()

- 编程步骤

  1. 继承Hive提供的类

     org.apache.hadoop.hive.ql.udf.generic.GenericUDF

     org.apache.hadoop.hive.ql.udf.generic.GenericUDTF

  2. 实现类中抽象方法

  3. 在hive命令行窗口创建函数

     ```shell
     # 添加jar
     add jar linux_jar_path
     # 创建function
     create [temporary] function [dbname.]function_name as class_name;
     ```

  4. 在hive命令行删除函数`drop [temporary] function [if exists] [dbname.]function_name`

### 自定义UDF

1. 编码，打包
2. ` add jar hive-learning-1.0-SNAPSHOT.jar `
3. `create temporary function my_len as "org.echisan.learning.hive.MyStringLength"`

```java
package org.echisan.learning.hive;

import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.exec.UDFArgumentTypeException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDF;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;

// 自定义一个 UDF 实现计算给定字符串的长度
public class MyStringLength extends GenericUDF {
    /**
     *
     * @param objectInspectors 输入参数类型的鉴别器对象
     * @return 返回值类型的鉴别器对象
     * @throws UDFArgumentException
     */

    @Override
    public ObjectInspector initialize(ObjectInspector[] objectInspectors) throws UDFArgumentException {
        if (objectInspectors.length != 1) {
            throw new UDFArgumentException("Input Args Length Error");
        }
        if (!objectInspectors[0].getCategory().equals(ObjectInspector.Category.PRIMITIVE)) {
            throw new UDFArgumentTypeException(0, "Input Args Type Error");
        }
        return PrimitiveObjectInspectorFactory.javaIntObjectInspector;
    }

    @Override
    public Object evaluate(DeferredObject[] deferredObjects) throws HiveException {
        if (deferredObjects[0].get() == null) {
            return 0;
        }

        return deferredObjects[0].get().toString().length();
    }

    @Override
    public String getDisplayString(String[] strings) {
        return "";
    }
}
```

### 自定义UDTF

1. `add jar my_udtf.jar`
2. `create temporary function myudtf as "org.echisan.learning.hive.MyUDTF";`
3. `select myudtf("hello,world,hadoop,hive",",");  `

```java
package org.echisan.learning.hive;

import org.apache.hadoop.hive.ql.exec.UDFArgumentException;
import org.apache.hadoop.hive.ql.metadata.HiveException;
import org.apache.hadoop.hive.ql.udf.generic.GenericUDTF;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.ObjectInspectorFactory;
import org.apache.hadoop.hive.serde2.objectinspector.StructObjectInspector;
import org.apache.hadoop.hive.serde2.objectinspector.primitive.PrimitiveObjectInspectorFactory;

import java.util.ArrayList;
import java.util.List;

public class MyUDTF extends GenericUDTF {
    private final List<String> outList = new ArrayList<>();

    @Override
    public StructObjectInspector initialize(StructObjectInspector argOIs) throws UDFArgumentException {
        // 1. 定义输出数据的列名和类型
        List<String> fieldNames = new ArrayList<>();
        List<ObjectInspector> fieldOIs = new ArrayList<>();
        // 2. 添加输出数据的列明和类型
        fieldNames.add("lineToWord");
        fieldOIs.add(PrimitiveObjectInspectorFactory.javaStringObjectInspector);
        return ObjectInspectorFactory.getStandardStructObjectInspector(
                fieldNames, fieldOIs
        );
    }

    @Override
    public void process(Object[] objects) throws HiveException {
        // 1. 获取原始数据
        String arg = objects[0].toString();
        // 2. 获取数据传入的第二个参数,此处为分隔符
        String splitKey = objects[1].toString();
        // 3. 将原始数据按照传入的分隔符进行切分
        String[] fields = arg.split(splitKey);
        // 4. 遍历切分后的结果，并写出
        for (String field : fields) {
            // 集合为复用的，首先清空集合
            outList.clear();
            //将每一个单词添加至集合
            outList.add(field);
            // 将集合内容写出
            forward(outList);
        }
    }

    @Override
    public void close() throws HiveException {

    }
}
```

# 压缩和存储

## hadoop压缩配置

### MR支持的压缩编码

![image-20220129114603795](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129114603795.png)

为了支持多种压缩/解压缩算法，Hadoop 引入了编码/解码器，如下表所示

![image-20220129114611414](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129114611414.png)

![image-20220129114622843](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129114622843.png)

压缩性能的比较

![image-20220129114635391](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129114635391.png)

### 压缩参数配置

要在 Hadoop 中启用压缩，可以配置如下参数（mapred-site.xml 文件中）：

![image-20220129114656780](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129114656780.png)

## 开启 Map 输出阶段压缩（MR 引擎）

开启 map 输出阶段压缩可以减少 job 中 map 和 Reduce task 间数据传输量。

实操

1. 开启hive中间件传输数据压缩功能

   `hive (default)>set hive.exec.compress.intermediate=true;`

2. 开启 mapreduce 中 map 输出压缩功能

   `hive (default)>set mapreduce.map.output.compress=true;`

3. 设置 mapreduce 中 map 输出数据的压缩方式

   `hive (default)>set mapreduce.map.output.compress.codec=org.apache.hadoop.io.compress.SnappyCodec;`

4. 执行查询语句

   `hive (default)> select count(ename) name from emp; `

## 开启 Reduce 输出阶段压缩

当 Hive 将 输 出 写 入 到 表 中 时 ， 输出内容同样可以进行压缩。属性 `hive.exec.compress.output`控制着这个功能。用户可能需要保持默认设置文件中的默认值false， 这样默认的输出就是非压缩的纯文本文件了。用户可以通过在查询语句或执行脚本中设置这 个值为 true，来开启输出结果压缩功能

实操

1. 开启 hive 最终输出数据压缩功能

   `hive (default)> set hive.exec.compress.output=true;`

2. 开启 mapreduce 最终输出数据压缩

   `set mapreduce.output.fileoutputformat.compress=true;`

3. 设置 mapreduce 最终数据输出压缩方式

   `set mapreduce.output.fileoutputformat.compress.codec =org.apache.hadoop.io.compress.SnappyCodec;`

4. 设置 mapreduce 最终数据输出压缩为块压缩

   `set mapreduce.output.fileoutputformat.compress.type=BLOCK;`

5. 测试一下输出结果是否是压缩文件

   `insert overwrite local directory '/home/echisan/bigdata/test-data/distribute-result-compress' select * from emp distribute by deptno sort by empno desc; `

## 文件格式存储

Hive 支持的存储数据的格式主要有：TEXTFILE 、SEQUENCEFILE、ORC、PARQUET。

### 列式存储和行式存储

![image-20220129131929901](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129131929901.png)

1. 行存储的特点

   查询满足条件的一整行数据的时候，列存储则需要去每个聚集的字段找到对应的每个列 的值，行存储只需要找到其中一个值，其余的值都在相邻地方，所以此时行存储查询的速度 更快。

2. 列存储的特点

   因为每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的
   数据量；每个字段的数据类型一定是相同的，列式存储可以针对性的设计更好的设计压缩算
   法。

   >TEXTFILE 和 SEQUENCEFILE 的存储格式都是基于行存储的； 
   >
   >ORC 和 PARQUET 是基于列式存储的。

#### TextFile

默认格式，数据不做压缩，磁盘开销大，数据解析开销大。可结合 Gzip、Bzip2 使用， 但使用 Gzip 这种方式，hive 不会对数据进行切分，从而无法对数据进行并行操作。

#### Orc

Orc (Optimized Row Columnar)是 Hive 0.11 版里引入的新的存储格式。

1. Index Data：一个轻量级的 index，默认是每隔 1W 行做一个索引。这里做的索引应该 只是记录某行的各字段在 Row Data 中的 offset。

2. Row Data：存的是具体的数据，先取部分行，然后对这些行按列进行存储。对每个 列进行了编码，分成多个 Stream 来存储。

3. Stripe Footer：存的是各个 Stream 的类型，长度等信息。

   每个文件有一个 File Footer，这里面存的是每个 Stripe 的行数，每个 Column 的数据类 型信息等；每个文件的尾部是一个 PostScript，这里面记录了整个文件的压缩类型以及 FileFooter 的长度信息等。在读取文件时，会 seek 到文件尾部读 PostScript，从里面解析到 File Footer 长度，再读 FileFooter，从里面解析到各个 Stripe 信息，再读各个 Stripe，即从后 往前读。

![image-20220129132322147](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129132322147.png)

#### Parquet

Parquet 文件是以二进制方式存储的，所以是不可以直接读取的，文件中包括该文件的 数据和元数据，**因此 Parquet 格式文件是自解析的。**

1. 行组(Row Group)：每一个行组包含一定的行数，在一个 HDFS 文件中至少存储一 个行组，类似于 orc 的 stripe 的概念。
2. 列块(Column Chunk)：在一个行组中每一列保存在一个列块中，行组中的所有列连 续的存储在这个行组文件中。一个列块中的值都是相同类型的，不同的列块可能使用不同的 算法进行压缩。
3. 页(Page)：每一个列块划分为多个页，一个页是最小的编码的单位，在同一个列块 的不同页可能使用不同的编码方式。

通常情况下，在存储 Parquet 数据的时候会按照 Block 大小设置行组的大小，由于一般 情况下每一个 Mapper 任务处理数据的最小单位是一个 Block，这样可以把**每一个行组由一 个 Mapper 任务处理**，增大任务执行并行度。

![image-20220129132637025](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129132637025.png)



### 对比

#### 存储大小对比

测试数据`19M Jan 29 13:36 log.data`

ORC`7.7 M` > Parquet`13.1 M` > textFile`18.1 M`

#### 查询速度对比

速度接近

```sql
insert overwrite local directory '/home/echisan/bigdata/test-data/log_text' select substring(url,1,4) from log_text;
Time taken: 1.31 seconds

insert overwrite local directory '/home/echisan/bigdata/test-data/log_orc' select substring(url,1,4) from log_orc;
Time taken: 1.336 seconds

insert overwrite local directory '/home/echisan/bigdata/test-data/log_parquet' select substring(url,1,4) from log_parquet;
Time taken: 1.344 seconds
```

### 存储方式和压缩总结

在实际的项目开发当中，hive 表的数据存储格式一般选择：orc 或 parquet。压缩方式一 般选择 snappy，lzo。

# 企业级调优

## 执行计划（Explain）

1. 基本语法

EXPLAIN [EXTENDED | DEPENDENCY | AUTHORIZATION] query

## Fetch 抓取

**Fetch 抓取是指，Hive 中对某些情况的查询可以不必使用 MapReduce 计算**。例如：SELECT * FROM employees;在这种情况下，Hive 可以简单地读取 employee 对应的存储目录下的文件， 然后输出查询结果到控制台。

```xml
<property>
 <name>hive.fetch.task.conversion</name>
 <value>more</value>
 <description>
 Expects one of [none, minimal, more].
 Some select queries can be converted to single FETCH task minimizing
latency.
 Currently the query should be single sourced not having any subquery
and should not have any aggregations or distincts (which incurs RS),
lateral views and joins.
 0. none : disable hive.fetch.task.conversion
 1. minimal : SELECT STAR, FILTER on partition columns, LIMIT only
 2. more : SELECT, FILTER, LIMIT only (support TABLESAMPLE and
virtual columns)
 </description>
</property>
```

## 本地模式

大多数的 Hadoop Job 是需要 Hadoop 提供的完整的可扩展性来处理大数据集的。不过， 有时 Hive 的输入数据量是非常小的。在这种情况下，为查询触发执行任务消耗的时间可能 会比实际 job 的执行时间要多的多。对于大多数这种情况，**Hive 可以通过本地模式在单台机 器上处理所有的任务。对于小数据集，执行时间可以明显被缩短**

用户可以通过设置 hive.exec.mode.local.auto 的值为 true，来让 Hive 在适当的时候自动 启动这个优化。

```
set hive.exec.mode.local.auto=true; //开启本地 mr
//设置 local mr 的最大输入数据量，当输入数据量小于这个值时采用 local mr 的方式，默认
为 134217728，即 128M
set hive.exec.mode.local.auto.inputbytes.max=50000000;
//设置 local mr 的最大输入文件个数，当输入文件个数小于这个值时采用 local mr 的方式，默
认为 4
set hive.exec.mode.local.auto.input.files.max=10;
```

## 表的优化

### 小表大表Join（MapJOIN）

新版的 hive 已经对小表 JOIN 大表和大表 JOIN 小表进行了优化。小表放 在左边和右边已经没有区别。

开启MapJoin参数设置

1. 设置自动选择Mapjoin;默认为 true

   `set hive.auto.convert.join = true; `

2. 大表小标阈值设置(默认25M以下认为是小表)

   `set hive.mapjoin.smalltable.filesize = 25000000;`

3. MapJoin工作机制

    ![image-20220129150056690](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129150056690.png)

### 大表join大表

#### 1. 空key过滤

有时 join 超时是因为某些 key 对应的数据太多，而相同 key 对应的数据都会发送到相同 的 reducer 上，从而导致内存不够。此时我们应该仔细分析这些异常的 key，很多情况下， 这些 key 对应的数据是异常数据，我们需要在 SQL 语句中进行过滤。

example

```sql
# 不过滤空id
hive (default)> insert overwrite table jointable select n.* from nullidtable n left join bigtable o on n.id=o.id;
Time taken: 11.827 seconds
# 过滤空id
hive (default)> insert overwrite table jointable select n.* from (select * from nullidtable where id is not null)  n left join bigtable o on n.id=o.id;
Time taken: 8.767 seconds
```

#### 2. 空key转换

有时虽然某个 key 为空对应的数据很多，但是相应的数据不是异常数据，必须要包含在 join 的结果中，此时我们可以表 a 中 key 为空的字段赋一个随机的值，使得数据随机均匀地 分不到不同的 reducer 上

#### 3. SMB（Sort Merge Bucket join）

// todo 没测出来提升了啥，反而更慢了

```shell
set hive.optimize.bucketmapjoin = true;
set hive.optimize.bucketmapjoin.sortedmerge = true;
set hive.input.format = org.apache.hadoop.hive.ql.io.BucketizedHiveInputFormat;
```

### Group By

默认情况下，Map 阶段同一 Key 数据分发给一个 reduce，当一个 key 数据过大时就倾斜 了。

![image-20220129161551730](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129161551730.png)

并不是所有的聚合操作都需要在 Reduce 端完成，很多聚合操作都可以先在 Map 端进行 部分聚合，最后在 Reduce 端得出最终结果。

**开启Map端聚合参数设置**

1. 是否在 Map 端进行聚合，默认为 True

   `set hive.map.aggr = true`

2. 在 Map 端进行聚合操作的条目数目

   `set hive.groupby.mapaggr.checkinterval = 100000`

3. 有数据倾斜的时候进行负载均衡（默认是 false）

   `set hive.groupby.skewindata = true`

   当选项设定为 true，生成的查询计划会有两个 MR Job。

   - 第一个 MR Job 中，Map 的输出 结果会随机分布到 Reduce 中，每个 Reduce 做部分聚合操作，并输出结果，这样处理的结果 是相同的 Group By Key 有可能被分发到不同的 Reduce 中，从而达到负载均衡的目的；
   - 第二 个 MR Job 再根据预处理的数据结果按照 Group By Key 分布到 Reduce 中（这个过程可以保证 相同的 Group By Key 被分布到同一个 Reduce 中），最后完成最终的聚合操作.

### Count(Distinct)去重统计

数据量小的时候无所谓，数据量大的情况下，由于 COUNT DISTINCT 操作需要用一个 Reduce Task 来完成，这一个 Reduce 需要处理的数据量太大，就会导致整个 Job 很难完成， 一般 COUNT DISTINCT 使用先 GROUP BY 再 COUNT 的方式替换,但是需要注意 group by 造成 的数据倾斜问题.

distinct

```sql
hive (default)> set mapreduce.job.reduces=5;
hive (default)> select count(distinct id) from bigtable;

Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 1
2022-01-29 16:26:55,884 Stage-1 map = 0%,  reduce = 0%
2022-01-29 16:27:01,028 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 4.05 sec
2022-01-29 16:27:06,136 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 7.14 sec
MapReduce Total cumulative CPU time: 7 seconds 140 msec
Ended Job = job_1643442541313_0001
MapReduce Jobs Launched: 
Stage-Stage-1: Map: 1  Reduce: 1   Cumulative CPU: 7.14 sec   HDFS Read: 129165946 HDFS Write: 105 SUCCESS
Total MapReduce CPU Time Spent: 7 seconds 140 msec
```

group by

```sql
hive (default)> select count(id) from (select id from bigtable group by id) a;

Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 5
2022-01-29 16:29:06,104 Stage-1 map = 0%,  reduce = 0%
2022-01-29 16:29:11,218 Stage-1 map = 100%,  reduce = 0%, Cumulative CPU 3.37 sec
2022-01-29 16:29:16,365 Stage-1 map = 100%,  reduce = 20%, Cumulative CPU 6.05 sec
2022-01-29 16:29:18,451 Stage-1 map = 100%,  reduce = 40%, Cumulative CPU 8.99 sec
2022-01-29 16:29:19,485 Stage-1 map = 100%,  reduce = 60%, Cumulative CPU 11.75 sec
2022-01-29 16:29:20,517 Stage-1 map = 100%,  reduce = 80%, Cumulative CPU 14.31 sec
2022-01-29 16:29:21,543 Stage-1 map = 100%,  reduce = 100%, Cumulative CPU 16.57 sec
MapReduce Total cumulative CPU time: 16 seconds 570 msec
Ended Job = job_1643442541313_0002
Launching Job 2 out of 2
Number of reduce tasks determined at compile time: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Starting Job = job_1643442541313_0003, Tracking URL = http://ubuntu:8088/proxy/application_1643442541313_0003/
Kill Command = /home/echisan/bigdata/hadoop-2.10.1/bin/hadoop job  -kill job_1643442541313_0003
Hadoop job information for Stage-2: number of mappers: 1; number of reducers: 1
2022-01-29 16:29:32,096 Stage-2 map = 0%,  reduce = 0%
2022-01-29 16:29:36,191 Stage-2 map = 100%,  reduce = 0%, Cumulative CPU 1.27 sec
2022-01-29 16:29:41,286 Stage-2 map = 100%,  reduce = 100%, Cumulative CPU 2.84 sec
MapReduce Total cumulative CPU time: 2 seconds 840 msec
Ended Job = job_1643442541313_0003
MapReduce Jobs Launched: 
Stage-Stage-1: Map: 1  Reduce: 5   Cumulative CPU: 16.57 sec   HDFS Read: 129178868 HDFS Write: 580 SUCCESS
Stage-Stage-2: Map: 1  Reduce: 1   Cumulative CPU: 2.84 sec   HDFS Read: 6631 HDFS Write: 105 SUCCESS
Total MapReduce CPU Time Spent: 19 seconds 410 msec
```

### 笛卡尔积

尽量避免笛卡尔积，join 的时候不加 on 条件，或者无效的 on 条件，Hive 只能使用 1 个 reducer 来完成笛卡尔积

### 行列过滤

列处理：在 SELECT 中，只拿需要的列，如果有分区，尽量使用分区过滤，少用 SELECT *。

 行处理：在分区剪裁中，当使用外关联时，如果将副表的过滤条件写在 Where 后面， 那么就会先全表关联，之后再过滤。

比如：

![image-20220129163422350](https://raw.githubusercontent.com/echisan/fiweofjaawef/main/img/image-20220129163422350.png)

### 分区
// todo
### 分桶
// todo

## 合理设置Map及Reduce数

1. 通常情况下，作业会通过 input 的目录产生一个或者多个 map 任务。 

   主要的决定因素有：input 的文件总个数，input 的文件大小，集群设置的文件块大小。

#### 复杂文件增加Map数

当 input 的文件都很大，任务逻辑复杂，map 执行非常慢的时候，可以考虑增加 Map 数， 来使得每个 map 处理的数据量减少，从而提高任务的执行效率。

增加 map 的方法为：根据公式， 调整 maxSize 最大值。让 maxSize 最大值低于 blocksize 就可以增加 map 的个数

```sql
computeSliteSize(Math.max(minSize,Math.min(maxSize,blocksize)))=blocksize=128M
```

设置最大切片值

```sql
hive (default)> set mapreduce.input.fileinputformat.split.maxsize=100;
```

#### 小文件进行合并

1. 在 map 执行前合并小文件，减少 map 数：`CombineHiveInputFormat` 具有对小文件进行合 并的功能（系统默认的格式）。`HiveInputFormat `没有对小文件合并功能。

```
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;
```

2. 在 Map-Reduce 的任务结束时合并小文件的设置：

   在 map-only 任务结束时合并小文件，默认 true

   `SET hive.merge.mapfiles = true;`

   在 map-reduce 任务结束时合并小文件，默认 false

   `SET hive.merge.mapredfiles = true;`

   合并文件的大小，默认 256M

   `SET hive.merge.size.per.task = 268435456;`

   当输出文件的平均大小小于该值时，启动一个独立的 map-reduce 任务进行文件 merge

   `SET hive.merge.smallfiles.avgsize = 16777216;`

### 合理设置Reduce数

#### 1. 调整 reduce 个数方法一

1. 每个 Reduce 处理的数据量默认是 256MB

   `hive.exec.reducers.bytes.per.reducer=256000000`

2. 每个任务最大的 reduce 数，默认为 1009

   `hive.exec.reducers.max=1009`

3. 计算 reducer 数的公式

   `N=min(参数 2，总输入数据量/参数 1)`

#### 2. 调整reduce个数方法二

在 hadoop 的 mapred-default.xml 文件中修改

设置每个 job 的 Reduce 个数

`set mapreduce.job.reduces = 15;`

#### 3. reduce 个数并不是越多越好

（1）过多的启动和初始化 reduce 也会消耗时间和资源； 

（2）另外，有多少个 reduce，就会有多少个输出文件，如果生成了很多个小文件，那 么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；

> 在设置 reduce 个数的时候也需要考虑这两个原则：处理大数据量利用合适的 reduce 数； 使单个 reduce 任务处理数据量大小要合适；

## 并行执行

Hive 会将一个查询转化成一个或者多个阶段。这样的阶段可以是 MapReduce 阶段、抽 样阶段、合并阶段、limit 阶段

默认情况下， Hive 一次只会执行一个阶段。不过，某个特定的 job 可能包含众多的阶段，而这些阶段可能 并非完全互相依赖的，也就是说有些阶段是可以并行执行的，这样可能使得整个 job 的执行 时间缩短。不过，如果有更多的阶段可以并行执行，那么 job 可能就越快完成。

通过设置参数 hive.exec.parallel 值为 true，就可以开启并发执行。不过，在共享集群中， 需要注意下，如果 job 中并行阶段增多，那么集群利用率就会增加。

```
set hive.exec.parallel=true; //打开任务并行执行
set hive.exec.parallel.thread.number=16; //同一个 sql 允许最大并行度，默认为8。
```

## 严格模式

Hive 可以通过设置防止一些危险操作：

1. 分区表不使用分区过滤

   将` hive.strict.checks.no.partition.filter` 设置为` true` 时，对于分区表，**除非 where 语句中含 有分区字段过滤条件来限制范围，否则不允许执行**。换句话说，就是用户不允许扫描所有分区。

2. 使用 order by 没有 limit 过滤

   将 `hive.strict.checks.orderby.no.limit `设置为 true 时，对于使用了 order by 语句的查询，要 求必须使用 limit 语句。因为 order by 为了执行排序过程会将所有的结果数据分发到同一个 Reducer 中进行处理，强制要求用户增加这个 LIMIT 语句可以防止 Reducer 额外执行很长一 段时间。

3. 笛卡尔积

   将 `hive.strict.checks.cartesian.product` 设置为 true 时，会限制笛卡尔积的查询。

## jvm重用

## 压缩

