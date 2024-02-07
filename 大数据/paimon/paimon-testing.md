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
