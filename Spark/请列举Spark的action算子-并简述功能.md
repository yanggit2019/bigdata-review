# 请列举Spark的action算子, 并简述功能

因为转换算子都是懒加载, 并不会立即执行, 行动算子是触发了整个作业的执.



**reduce(f: (T, T) => T**

f函数聚集 RDD 中的所有元素, 先聚合分区内数据, 再聚合分区间数据.



**collect**

在驱动程序中, 以数组 Array 的形式返回数据集的所有元素.



**first**

返回 RDD 中的第一个元素.



**take(num :Int)**

返回一个由 RDD 的前 n 个元素组成的数组.



**aggregate**

将分区内的元素通过分区内的逻辑和初始值进行聚合, 然后利用分区间的逻辑和初始值进行操作.

注意, 分区间逻辑再次使用初始值, 这个和 aggregateByKey 是有区别的.



**countByKey**

统计每种 key 的个数.



**foreach**

遍历 RDD 中的每一个元素, 并依次应用 f 函数.



**saveAsTextFile**

将数据集的元素以 textfile 的形式保存到 HDFS 文件系统或者其他支持的文件系统, 对于每个元素, Spark 将会调用 toString 方法, 将它装换为文件中的文本.