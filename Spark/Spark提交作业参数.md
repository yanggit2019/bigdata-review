# Spark提交作业参数

在提交任务时的几个重要参数:

`driver-cores` :  driver 使用内核数, 默认为 1.

`driver-memory` : driver 内存大小, 默认 512 M.

`executor-cores` : 每个 Executor 使用的内核数, 默认为 1, 官方建议 2 - 5 个, 我们企业是 4 个.

`num-executors` : 启动 Executors 的数量, 默认为 2.

`executor-memory` : Executor 内存大小, 默认 1 G.



给一个提交任务的样式

```shell
spark-submit \
  --master local[5]  \
  --driver-cores 2   \
  --driver-memory 8g \
  --executor-cores 4 \
  --num-executors 10 \
  --executor-memory 8g \
  --class PackageName.ClassName XXXX.jar \
  --name "Spark Job Name" \
  InputPath      \
  OutputPath
```



# Refences

1. [Spark提交参数说明和常见优化](https://blog.csdn.net/gamer_gyt/article/details/79135118)

