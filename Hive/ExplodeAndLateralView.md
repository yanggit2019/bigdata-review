# Explode and Lateral view

## Explode

- UDTF
- 输入: List 或 Map
- 输出: 多行
- 作用: 一列转为多行

例子:

```sql
create table t1 (id int,name string)

insert into t1 (id,name) values (1,'a,b,c'),(2,'d,e,f')

SELECT  explode(split(t.name,',')) from t1 t
```

输出:

```
explode_col
a
b
c
d
e
f
```

顾名思义, 这个函数是爆炸的意思, 当然就是把一列数据的每一行变成多行数据, 那么输入应该是 List 或 Map 结构, 能产生这个 List 结构的比如 `collect_list/collect_map`. 也常用 `explode(split(string_field, ','))`.

另外注意, 这里是直接用在 select 后的, 也就是从一个表列转行然后直接获取结果. 但我们一般需要配合原表的其他字段进行使用, 这个时候就会用到 `lateral view`

## Lateral view

explode 可以**一列中的每一行拆分为多行**, 但如果想连接原来的字段, 就需要 lateral view.

```sql
from table_name lateral view UDTF(expression) tableAliasName as colAliasName
```

- lateral view 用在 from table_name 之后;
- 需要一个 UDTF 用来行转列, 之后原表字段会和 UDTF 之后的字段连接成一个虚表, 这张虚表就包含原来的字段和 UDTF 增加的字段;
- 虚表的名称叫 tableAliasName, 需要指定;
- UDTF 之后的列名叫做 colAliasName, 需要指定;
- 可以在 select 语句中选择原来的字段或 UDTF 生成的字段.

```sql
SELECT myTab.id, myTab.sq, myTab.myCol 
from window_test_table 
LATERAL VIEW explode(split(sq,',')) myTab as myCol;
```

结果如下:

```
id, sq, mycol
1,"a,b,c",a
1,"a,b,c",b
1,"a,b,c",c
2,"c,d,e",c
2,"c,d,e",d
2,"c,d,e",e
```

分析执行计划:

```sql
explain SELECT id, sq,myCol
from window_test_table
LATERAL VIEW explode(split(sq,',')) myTab as myCol;
```

**结果见文 3**, 有以下几个注意点:

1. outer 关键字可以把不输出 UDTF 的空结果, 输出成 Null, 默认是不输出, 从而防止数据丢失;
2. lateral view explode 并不会产生 shuffle, 执行计划中 LateralViewJoinOperator 处理逻辑也是很简单明了, 这里的 join 也是简单的 List.addAll, join 代表的是两份数据联接到一起的意思, 并不是真正的意义上的 join. 况且, 从执行任务来看, 也没有 reduce 任务, 所以必然没有 shuffle.
3. 这里经过测试发现使用 MR 或 Spark 计算引擎的执行计划是一样的.

# References

1. [Hive中explode和lateral view组合的用法](https://blog.csdn.net/lyzx_in_csdn/article/details/85628867)

2. [hive lateral view 与 explode详解](https://blog.csdn.net/bitcarmanlee/article/details/51926530)
3. [你真的了解Lateral View explode吗？--源码复盘](https://zhuanlan.zhihu.com/p/137482744)

4. [Manual Function - Explode](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF#LanguageManualUDF-explode)

