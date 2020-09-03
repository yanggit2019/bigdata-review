# Group by or Count(distinct)

Hive 中去重一般有两种实现方式:

**方式一**

在 mysql 中常用这种方式.

```sql
select count(distinct s_age)
from student_tb_orc
```

Hive 中的执行任务:

总共一个阶段, map 端在内存中构建 hashtable 进行去重, 然后再 shuffle, 所有数据进入一个 reducer 再根据 去重字段计数.

**方式二**

hive 中对大枚举值字段去重常用的方法.

```sql
select count(1) from(
    select s_age
    from student_tb_orc
    group by s_age
) b
```

Hive 中的执行任务:

总共两个阶段, 第一个阶段通过 shuffle 将去重字段分组, 然后传入到若干个 (可以手动设定) reducer 中. 第二阶段再在 map 端去重, 最后 shuffle 进入一个 reducer 进行计数.



# 选择

需要根据字段枚举数量大小进行选择.

## 枚举数量大

通常所谓的数据量特别大的情况下能够有效避免 reducer 端的数据倾斜, 就是针对这种情况.

比如几亿条数据, 数据枚举量大.

`count (distinct field_name)` 

1. 每一个 map 端在去重时, distinct 的命令会在内存中构建一个 hashtable, 过多的数据就有可能导致 map 端 OOM.
2. 如果枚举数量大, 进入 reducer 数据量也会很大, 且所有 map 的数据都进入一个 reducer 端, 进行排序, 很容易就导致数据倾斜.

`group by`

第一阶段按照字段分区, 可以自己指定 reducer 的个数, 这样即使数据量再大, 但并行度就提高了. 

## 枚举数量小

这个时候反而 `count distinct` 效率要高, 因为 枚举量少意味着在 map 阶段去重时并不会出现 OOM, 同样高效去重后, 到达 reducer 内的数据会少很多, 完全不用担心数据倾斜.





# References

1. [*一篇文章让你了解Hive调优](https://cloud.tencent.com/developer/article/1591607)
2. [*HIVE group by 和count(distinct)进行对比](https://blog.csdn.net/u013385925/article/details/77484147)
3. [Why is count(distinct) slower than group by in Hive?](https://stackoverflow.com/questions/19311193/why-is-countdistinct-slower-than-group-by-in-hive)
4. [Hive之COUNT DISTINCT优化](https://datavalley.github.io/2016/02/15/Hive%E4%B9%8BCOUNT-DISTINCT%E4%BC%98%E5%8C%96)
5. [hive count distinct和group by ](https://my.oschina.net/u/2000675/blog/2989271)
6. [Hive SQL优化之 Count Distinct](https://www.jianshu.com/p/dae10bce7279)

