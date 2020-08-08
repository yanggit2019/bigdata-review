# Spark有几种部署方式

部署 Spark 集群大体上分为两种模式: 单机模式与集群模式.

大多数分布式框架都支持单机模式, 方便开发者调试框架的运行环境. 但是在生产环境中, 并不会使用单机模式. 

1. Local: 运行在一台机器上, 通常是练手或者测试环境.
2. Standalone: 构建一个基于 Master + Slaves 的资源调度集群, Spark 任务提交给 Master 运行. 这是 Spark 自带的任务调度模式 (国内常用).
3. Yarn: Spark 客户端直接连接 Yarn, 不需要额外构建 Spark 集群. 有 yarn-client 和 yarn-cluster 两种模式, 主要区别在于 Driver 程序的运行节点 (国内常用).
4. Mesos：Spark 使用 Mesos 平台进行资源与任务的调度, 国内大环境比较少用.

