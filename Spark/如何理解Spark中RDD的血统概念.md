# 如何理解Spark中RDD的血统概念

RDD 在 Lineage 依赖方面分为两种: Narrow Dependencies 与 Wide Dependencies. 

用来解决数据容错时的高效性, 在划分任务时候也起到重要作用.