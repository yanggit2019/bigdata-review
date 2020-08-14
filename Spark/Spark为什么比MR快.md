# Spark 为什么比 MR 快

通常大家只是说 Spark 是基于内存计算的, 速度比 MapReduce 要快. 或者说内存中迭代计算. 其实我们要抓住问题的本质. 总结有以下几点: 

## DAG 相比 MR 在大多数情况下可以减少 Shuffle 次数

Spark vs MapReduce ≠ 内存 vs 磁盘.

其实 Spark 和 MapReduce 的计算都发生在内存中, 只不过 Spark 支持将需要反复用到的数据给 Cache 到内存中, 减少数据加载耗时, 所以 Spark 跑机器学习算法比较在行 (需要对数据进行反复迭代), Spark 基于磁盘的计算依然也是比 Hadoop 快.

区别在于: MapReduce 通常需要将计算的中间结果写入磁盘, 然后还要读取磁盘, 从而导致了频繁的磁盘 IO. Spark 则不需要将计算的中间结果写入磁盘, 这得益于 Spark 的 RDD (弹性分布式数据集, 很强大) 和 DAG (有向无环图) , 其中 DAG 记录了 job 的 stage 以及在 job 执行过程中父 RDD 和子 RDD 之间的依赖关系, Spark 的 DAGScheduler 相当于一个改进版的 MapReduce, 如果计算不涉及与其他节点进行数据交换, Spark 可以在内存中一次性完成这些操作, 也就是中间结果无须落盘, 以 RDD 的形式存放在内存中, 且能够从 DAG 中恢复, 大大减少了磁盘 IO. 

但是, 如果计算过程中涉及数据交换, Spark 也是会把 shuffle 的数据写磁盘的.

## Spark vs MapReduce Shuffle的不同

Spark 和 MapReduce 在计算过程中通常都不可避免的会进行 Shuffle, 两者至少有一点不同: MapReduce 在 Shuffle 时需要花费大量时间进行排序, 排序在 MapReduce 的 Shuffle 中似乎是不可避免的；Spark 在 Shuffle 时则只有部分场景才需要排序, 支持基于 Hash 的分布式聚合, 更加省时；

## 多进程模型 vs 多线程模型的区别

MapReduce 采用了多进程模型, 而 Spark 采用了多线程模型. 多进程模型的好处是便于**细粒度控制每个任务占用的资源**, 但每次任务的启动都会消耗一定的启动时间. 就是说 MapReduce 的 Map Task 和 Reduce Task 是进程级别的, 而 Spark Task 则是基于线程模型的, 就是说 Mapreduce 中的 map 和 reduce 都是 jvm 进程, 每次启动都需要**重新申请资源**, 消耗了不必要的时间 (假设容器启动时间大概 1s, 如果有 1200 个 block, 那么单独启动 map 进程事件就需要 20 分钟) Spark 则是通过**复用线程池中的线程**来减少启动、关闭 task 所需要的开销.  (多线程模型也有缺点, 由于同节点上所有任务运行在一个进程中, 因此, 会出现严重的资源争用, 难以细粒度控制每个任务占用资源) 

# References

1. [为什么Spark比MapReduce快？ - 大数据技术架构的回答 - 知乎](https://www.zhihu.com/question/31930662/answer/1247877997)
2. [MapReduce Shuffle 和 Spark Shuffle 结业篇](https://mp.weixin.qq.com/s?__biz=MzUxOTU5Mjk2OA==&mid=2247485991&idx=1&sn=79c9370801739813b4a624ae6fa55d6c&chksm=f9f60740ce818e56a18f8782d21d376d027928e434f065ac2c251df09d2d4283710679364639&scene=21#wechat_redirect)
3. [为什么Spark比MapReduce快？ - 翟士丹的回答 - 知乎](https://www.zhihu.com/question/31930662/answer/151036900)
