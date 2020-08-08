# 简述Spark的宽窄依赖, 以及Spark如何划分stage, 每个stage又根据什么决定task个数

RDD 和它依赖的父 RDD(s) 的关系有两种不同的类型, 即窄依赖 (narrow dependency) 和宽依赖 (wide dependency) .

窄依赖表示每一个父 RDD 的 Partition 最多被子 RDD 的一个 Partition 使用.

宽依赖表示同一个父 RDD 的 Partition 被多个子 RDD 的 Partition 依赖, 会引起 Shuffle.

具有宽依赖的 **transformations** 包括: **sort**, **reduceByKey**, **groupByKey**, **join**, 和调用 **rePartition** 函数的任何操作.

宽依赖对 Spark 去评估一个 transformations 有更加重要的影响, 比如对性能的影响.

**Stage**: 根据 RDD 之间的依赖关系的不同将 Job 划分成不同的 Stage, 遇到一个宽依赖则划分一个 Stage.

**Task**: Stage 是一个TaskSet, 根据该 Stage 的最后一个 RDD 的分区数划分成一个个的 Task.

