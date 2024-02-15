# Paimon Testing

install hms

```bash
docker run -d -p 9083:9083 --env SERVICE_NAME=metastore \
 --name metastore-standalone  \
 apache/hive:3.1.3
```

## Testing lookup changelog-producer

```sql

CREATE CATALOG paimon_local_catalog WITH (
     'type'='paimon',
     'warehouse'='file:///tmp/paimon'
 );

use catalog paimon_local_catalog;
    
create table T (
    id int,
    name varchar,
  primary key (`id`) not enforced
)
with(
    'changelog-producer' = 'lookup'
  );
set 'execution.runtime-mode' = 'batch';
insert into T values (1, 'hello'),
                     (2, 'world'),
                     (3, 'huawei'),
                     (4, 'hello'),
                     (5, 'world'),
                     (6, 'world'),
                     (7, 'huawei');

create table CNT (
    name varchar primary key ,
    cnt bigint
)
with(
    'changelog-producer' = 'lookup'
    );
   
-- console2
set 'execution.checkpointing.interval' = '10s';
insert into CNT
    select name, count(*) cnt
    from T group by name;

--- console3
insert into T values (8, 'hello');
```

测试lookup生成changelog
- console-1 streaming query paimon CNT表
- console-2 streaming insert paimon CNT表select count(*) from T
- console-3 batch insert paimon T 表

最终console-1输出了`-U`,`+U`的记录
```bash
Flink SQL> select * from CNT;
+----+--------------------------------+----------------------+
| op |                           name |                  cnt |
+----+--------------------------------+----------------------+
| +U |                          hello |                    3 |
| +U |                         huawei |                    2 |
| +U |                          world |                    4 |
| -U |                          world |                    4 |
| +U |                          world |                    5 |
```

## 测试paimon源表消费点位
通过配置 `scan.mode` 即可， 总共有几项配置可以配置
- default
    - 如果设置了`scan.timestamp-millis`，则行为与from-timestamp参数值相同。
    - 如果设置了`scan.snapshot-id`，则行为与from-snapshot参数值相同；
    - 如果以上两个参数都没有设置，则行为与latest-full参数值相同。

### default

```sql
Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='default')*/;
+----+--------------------------------+----------------------+
| op |                           name |                  cnt |
+----+--------------------------------+----------------------+
| +U |                          hello |                    3 |
| +U |                         huawei |                    2 |
| +U |                          world |                    5 |
^CQuery terminated, received a total of 3 rows
```

### latest-full

- batch: 产出表的最新snapshot。
- streaming: 作业启动时，首先产出表的最新snapshot，之后持续产出增量数据。

```sql
Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='latest-full')*/;
+----+--------------------------------+----------------------+
| op |                           name |                  cnt |
+----+--------------------------------+----------------------+
| +U |                          hello |                    3 |
| +U |                         huawei |                    2 |
| +U |                          world |                    5 |
^CQuery terminated, received a total of 3 rows
```

### compacted-full
- batch: 产出表最近一次compact后的snapshot。
- streaming: 作业启动时，首先产出表最近一次compact后的snapshot，之后持续产出增量数据。

```sql

Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='compacted-full')*/;
+----+--------------------------------+----------------------+
| op |                           name |                  cnt |
+----+--------------------------------+----------------------+
| +U |                          hello |                    3 |
| +U |                         huawei |                    2 |
| +U |                          world |                    5 |
^CQuery terminated, received a total of 3 rows
```

### latest
- batch: 与latest-full相同。
- streaming: 作业启动时不产出表的最新snapshot，之后持续产出增量数据。

```sql
Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='latest')*/;
+----+--------------------------------+----------------------+
| op |                           name |                  cnt |
+----+--------------------------------+----------------------+
^CQuery terminated, received a total of 0 row
```

### from-timestamp
- batch: 产出表在scan.timestamp-millis之前（含）的最新snapshot。
- streaming: 作业启动时不产出snapshot，之后持续产出从scan.timestamp-millis开始（含）的增量数据。

```sql
-- 指定了snapshot-1文件里的时间戳
-- streaming
Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='from-timestamp', 'scan.timestamp-millis'='1707283513008' */;
+----+--------------------------------+----------------------+
| op |                           name |                  cnt |
+----+--------------------------------+----------------------+
| +I |                          hello |                    2 |
| +I |                         huawei |                    2 |
| +I |                          world |                    3 |
| -U |                          hello |                    2 |
| +U |                          hello |                    3 |
| -U |                         huawei |                    2 |
| +U |                         huawei |                    2 |
| -U |                          world |                    3 |
| +U |                          world |                    3 |
| -U |                          world |                    3 |
| +U |                          world |                    4 |
| -U |                          world |                    4 |
| +U |                          world |                    5 |

-- batch
set 'execution.runtime-mode' = 'batch';
Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='from-timestamp', 'scan.timestamp-millis'='1707283513008')*/;
+--------+-----+
|   name | cnt |
+--------+-----+
|  hello |   2 |
| huawei |   2 |
|  world |   3 |
+--------+-----+


```

### from-snapshot

- batch: 产出表的snapshot，snapshot编号由scan.snapshot-id指定。
- streaming: 作业启动时不产出snapshot，之后持续产出从scan.snapshot-id开始（含）的增量数据。


```sql
-- batch
select * from CNT/*+OPTIONS('scan.mode'='from-snapshot', 'scan.snapshot-id'='1') */
    > ;
+--------+-----+
|   name | cnt |
+--------+-----+
|  hello |   2 |
| huawei |   2 |
|  world |   3 |
+--------+-----+
3 rows in set

Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='from-snapshot', 'scan.snapshot-id'='2') */
    > ;
+--------+-----+
|   name | cnt |
+--------+-----+
|  hello |   2 |
| huawei |   2 |
|  world |   3 |
+--------+-----+
3 rows in set

Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='from-snapshot', 'scan.snapshot-id'='3') */
    > ;
+--------+-----+
|   name | cnt |
+--------+-----+
|  hello |   3 |
| huawei |   2 |
|  world |   3 |
+--------+-----+
3 rows in set

Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='from-snapshot', 'scan.snapshot-id'='4') */
    > ;
+--------+-----+
|   name | cnt |
+--------+-----+
|  hello |   3 |
| huawei |   2 |
|  world |   3 |
+--------+-----+
3 rows in set

--streaming
set 'execution.runtime-mode' = 'streaming';
Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='from-snapshot', 'scan.snapshot-id'='1') */;
+----+--------------------------------+----------------------+
| op |                           name |                  cnt |
+----+--------------------------------+----------------------+
| +I |                          hello |                    2 |
| +I |                         huawei |                    2 |
| +I |                          world |                    3 |
| -U |                          hello |                    2 |
| +U |                          hello |                    3 |
| -U |                         huawei |                    2 |
| +U |                         huawei |                    2 |
| -U |                          world |                    3 |
| +U |                          world |                    3 |
| -U |                          world |                    3 |
| +U |                          world |                    4 |
| -U |                          world |                    4 |
| +U |                          world |                    5 |
^CQuery terminated, received a total of 13 rows

```

### from-snapshot-full
- batch: 与from-snapshot相同。
- streaming: 作业启动时产出表的snapshot，snapshot编号由scan.snapshot-id指定，
  之后持续产出从scan.snapshot-id开始（不含）的增量数据。

```sql
Flink SQL> select * from CNT/*+OPTIONS('scan.mode'='from-snapshot-full', 'scan.snapshot-id'='1') */;
+----+--------------------------------+----------------------+
| op |                           name |                  cnt |
+----+--------------------------------+----------------------+
| +I |                          hello |                    2 |
| +I |                         huawei |                    2 |
| +I |                          world |                    3 |
| -U |                          hello |                    2 |
| +U |                          hello |                    3 |
| -U |                         huawei |                    2 |
| +U |                         huawei |                    2 |
| -U |                          world |                    3 |
| +U |                          world |                    3 |
| -U |                          world |                    3 |
| +U |                          world |                    4 |
| -U |                          world |                    4 |
| +U |                          world |                    5 |
^CQuery terminated, received a total of 13 rows
```


