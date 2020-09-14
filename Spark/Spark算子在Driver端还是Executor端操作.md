# Spark 算子在 Driver 端还是 Executor 端操作

对 RDD 具体数据的操作都是在 Executor 上执行的, 所有对 RDD 自身的操作都是在 Driver 上执行的.

`foreach` 和 `freachPartition` 是 RDD 的计算方法, 所以是对具体数据的计算操作;

`foreachRDD` 中传入的函数是 对 RDD 进行操作的函数, 不是操作具体数据的, 所以是在 Driver 端的.

所以说, `foreachRDD` 传入的对 RDD 进行操作的函数如果 `RDD.asInstanceOf[HasOffsetRanges].offsetRanges` 并不触发计算的, 那就意味着此段代码只在 Driver 端进行.

如果像传入的函数是 `RDD.map/foreach/foreachPartition`, 这些设计 RDD 内数据具体计算的, 那就是要在 Executor 端进行了, 这个时候很可能就出现给 `foreachRDD()` 传入的函数既设计到在 Driver 端执行的代码, 又有在 Executor 端计算的 RDD 算子, 所以就涉及到一个序列化的问题.



# 官方示例

**Design Patterns for using foreachRDD**

dstream.foreachRDD is a powerful primitive that allows data to be sent out to external systems. However, it is important to understand how to use this primitive correctly and efficiently. Some of the common mistakes to avoid are as follows.

Often writing data to external system requires creating a connection object (e.g. TCP connection to a remote server) and using it to send data to a remote system. For this purpose, a developer may inadvertently try creating a connection object at the Spark driver, and then try to use it in a Spark worker to save records in the RDDs. For example (in Scala),

```scala
dstream.foreachRDD { rdd =>
  val connection = createNewConnection()  // executed at the driver
  rdd.foreach { record =>
    connection.send(record) // executed at the worker
  }
}
```

This is incorrect as this requires the connection object to be serialized and sent from the driver to the worker. Such connection objects are rarely transferable across machines. This error may manifest as serialization errors (connection object not serializable), initialization errors (connection object needs to be initialized at the workers), etc. The correct solution is to create the connection object at the worker.

This is incorrect as this requires the connection object to be serialized and sent from the driver to the worker. Such connection objects are rarely transferable across machines. This error may manifest as serialization errors (connection object not serializable), initialization errors (connection object needs to be initialized at the workers), etc. The correct solution is to create the connection object at the worker.

However, this can lead to another common mistake - creating a new connection for every record. For example,

```scala
dstream.foreachRDD { rdd =>
  rdd.foreach { record =>
    val connection = createNewConnection()
    connection.send(record)
    connection.close()
  }
}
```

Typically, creating a connection object has time and resource overheads. Therefore, creating and destroying a connection object for each record can incur unnecessarily high overheads and can significantly reduce the overall throughput of the system. A better solution is to use rdd.foreachPartition - create a single connection object and send all the records in a RDD partition using that connection.

Typically, creating a connection object has time and resource overheads. Therefore, creating and destroying a connection object for each record can incur unnecessarily high overheads and can significantly reduce the overall throughput of the system. A better solution is to use rdd.foreachPartition - create a single connection object and send all the records in a RDD partition using that connection.

```scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    val connection = createNewConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    connection.close()
  }
}
```

This amortizes the connection creation overheads over many records.

This amortizes the connection creation overheads over many records.

Finally, this can be further optimized by reusing connection objects across multiple RDDs/batches. One can maintain a static pool of connection objects than can be reused as RDDs of multiple batches are pushed to the external system, thus further reducing the overheads.

```scala
dstream.foreachRDD { rdd =>
  rdd.foreachPartition { partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

Note that the connections in the pool should be lazily created on demand and timed out if not used for a while. This achieves the most efficient sending of data to external systems.

Note that the connections in the pool should be lazily created on demand and timed out if not used for a while. This achieves the most efficient sending of data to external systems.

Other points to remember:

- DStreams are executed lazily by the output operations, just like RDDs are lazily executed by RDD actions. Specifically, RDD actions inside the DStream output operations force the processing of the received data. Hence, if your application does not have any output operation, or has output operations like dstream.foreachRDD() without any RDD action inside them, then nothing will get executed. The system will simply receive the data and discard it.

- By default, output operations are executed one-at-a-time. And they are executed in the order they are defined in the application.



# References

1. [解惑| spark实现业务前一定要掌握的点](https://zhuanlan.zhihu.com/p/94346878)
2. [http://spark.apache.org/](http://spark.apache.org/docs/2.4.0/streaming-programming-guide.html)
