# Flink

flink引擎下使用paimon

# Savepoint and recover
因为paimon有自己的snapshot管理，这可能跟flink的checkpoint管理存在冲突导致paimon
从savepoint中恢复时出现异常，但也不用担心，这不会导致存储被破坏的问题。

推荐使用下面的方式去触发savepoint
- 使用Stop with savepoint
- 使用[Tag with savepoint](todo), 然后从savepoint恢复之前先rollback-to-tag


# 支持的Flink数据类型

不支持的类型如下，其他的都支持
- MULTISET
- 主键不支持MAP类型




