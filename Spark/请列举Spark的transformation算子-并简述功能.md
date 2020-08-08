# 请列举Spark的transformation算子-并简述功能

`map(func)`

返回一个新的 RDD, 该 RDD 由每一个输入元素经过 func 函数转换后组成.

`mapPartitions(func)`

类似于 map, 但独立地在 RDD 的每一个分区上运行, 因此在类型为 T 的 RDD 上运行时, func 的函数类型必须是 Iterator[T] => Iterator[U]. 假设有 N 个元素, 有 M 个分区, 那么 map 的函数的将被调用 N 次,而 mapPartitions 被调用 M 次,一个函数一次处理所有分区. 

mapPartition() 每次处理一个分区的数据, 这个分区的数据处理完后, 原 RDD 中分区的数据才会释放, 可能导致 OOM.

`reduceByKey(func: (V,V) => V, [numTask])`

在一个 kv 的 RDD 上调用, 返回一个 kv 的RDD, 使用指定的 reduce 函数, 将相同 key 的值聚合到一起, reduce 任务的个数可以通过第二个可选的参数来设置. 

`aggregateByKey (zeroValue:U,[partitioner: Partitioner]) (seqOp: (U, V) => U,combOp: (U, U) => U)`

此函数作用于 kv 类型的 RDD, 首先给定一个 v 的初值 U, 然后每个分区内以 k 分组按照 seqOP 进行聚合, 聚合后不同分区的 kv 值进行 shuffle, shuffle 后将分区间同一个 k 对应的 v 按照 combOp 进行聚合. 最后将 k 与计算结果作为一个新的 kv 对输出. 

`combineByKey(createCombiner: V=>C, mergeValue: (C, V) =>C, mergeCombiners: (C, C) =>C)`

按照 k 分组后, 会把第一个值 v 利用 `createCombiner` 函数变成特定的结构 c , 然后特定结构 c 和下一个 v 按照 `mergeValue: (C, V) =>C` 函数得到新的 c 值并以此类推, 最后得到 k-c 键值对.

然后进行 shuffule, 不同分区间的相同 k 的 c 值再按照 `mergeCombiners: (C, C) =>C)` 函数进行聚合. 最终返回 kc 键值对.

1. `createCombiner: combineByKey()` 会遍历分区中的所有元素, 因此每个元素的键要么还没有遇到过, 要么就和之前的某个元素的键相同. 如果这是一个新的元素, combineByKey()会使用一个叫作createCombiner()的函数来创建那个键对应的累加器的初始值.

2. `mergeValue`: 如果这是一个在处理当前分区之前已经遇到的键, 它会使用mergeValue()方法将该键的累加器对应的当前值与这个新的值进行合并

3. `mergeCombiners`: 由于每个分区都是独立处理的,  因此对于同一个键可以有多个累加器. 如果有两个或者更多的分区都有对应同一个键的累加器,  就需要使用用户提供的 mergeCombiners() 方法将各个分区的结果进行合并. 

