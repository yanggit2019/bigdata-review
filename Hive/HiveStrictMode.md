# Hive 严格模式

Hive提供了一个严格模式, 可以防止用户执行那些可能产生意向不到的不好的效果的查询. 说通俗一点就是这种模式可以阻止某些查询的执行. 通过如下语句设置严格模式: 

```java
hive> set hive.mapred.mode=strict;
```

**设置为严格模式后, 可以禁止3种类型的查询**: 

## 带有分区的表的查询

如果在一个分区表执行hive, 除非where语句中包含分区字段过滤条件来显示数据范围, 否则不允许执行. 换句话说就是在严格模式下不允许用户扫描所有的分区. 进行这个限制的原因是, 通常分区表都拥有非常大的数据集, 而且数据增加迅速. 如果不进行分区限制的查询会消耗巨大的资源来处理, 如下不带分区的查询语句: 

```java
hive> SELECT DISTINCT(planner_id) FROM fracture_ins WHERE planner_id=5;
```

执行后会出现如下错误: 

```java
FAILED: Error in semantic analysis: No Partition Predicate Found for Alias "fracture_ins" Table "fracture_ins
```

解决方案是在 where 中增加分区条件: 

```java
hive> SELECT DISTINCT(planner_id) FROM fracture_ins
    > WHERE planner_id=5 AND hit_date=20120101;
```

## 带有 order by 的查询

对于使用了 order by 的查询, 要求必须有 limit 语句. 因为 order by 为了执行排序过程会将所有的结果分发到同一个 reduce 中进行处理, 强制要求用户增加这个 limit 语句可以防止 reduce 额外消耗资源, 如下是不带 limit 关键字的查询语句: 

```java
hive> SELECT * FROM fracture_ins WHERE hit_date>2012 ORDER BY planner_id;
```

出现如下错误: 

```java
FAILED: Error in semantic analysis: line 1:56 In strict mode,
limit must be specified if ORDER BY is present planner_id
```

解决方案就是增加一个limit关键字: 

```java
hive> SELECT * FROM fracture_ins WHERE hit_date>2012 ORDER BY planner_id
    > LIMIT 100000;
```

## 限制笛卡尔积的查询

对关系型数据库非常了解的用户可能期望在执行 join 查询的时候不使用 on 语句而是使用 where 语句, 这样关系型数据库的执行优化器就可以高效的将 where 语句转换成那个 on 语句了. 不幸的是, Hive 并不支持这样的优化, 因为如果表非常大的话, 就会出现不可控的情况, 如下是不带 on 的语句: 

```java
hive> SELECT * FROM fracture_act JOIN fracture_ads
    > WHERE fracture_act.planner_id = fracture_ads.planner_id;
```

出现如下错误: 

```java
FAILED: Error in semantic analysis: In strict mode, cartesian product
is not allowed. If you really want to perform the operation,
+set hive.mapred.mode=nonstrict+
```

解决方案就是加上on语句: 

```java
hive> SELECT * FROM fracture_act JOIN fracture_ads
    > ON (fracture_act.planner_id = fracture_ads.planner_id);
```

