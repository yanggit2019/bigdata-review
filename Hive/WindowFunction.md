# Window Function

## hql 执行顺序

```sql
-- sql 执行顺序
from → where → group by → having → select → order by 
-- hql 执行顺序
from → where → select → group by → having → order by
```

## 开窗范围

```
(ROWS | RANGE) BETWEEN (UNBOUNDED | [num]) PRECEDING AND ([num] PRECEDING | CURRENT ROW | (UNBOUNDED | [num]) FOLLOWING)
(ROWS | RANGE) BETWEEN CURRENT ROW AND (CURRENT ROW | (UNBOUNDED | [num]) FOLLOWING)
(ROWS | RANGE) BETWEEN [num] FOLLOWING AND (UNBOUNDED | [num]) FOLLOWING
```

## 语法

语法形式: 

```sql
select UDAF(expression) over(partition by field1 order by field2 开窗范围)
```

说明: 

- 窗口函数一般本身带有分区 (分组) 功能, 所以不同于 `group by`  + 聚合函数, 不需要 `group by` 意味着, 如果没有 where 条件过滤, 使用窗口函数后得到的结果行数是不变的, 只是在每一行后添加计算的一列;
- `partition by` 用于分区 (分组), 如果不加 `order by`, 则默认对于每一行, 开窗的范围是**其所在的分区**;
- 一旦加了 `order by`, 则首先默认开窗范围为所在分区内, **第一行到当前行**;
- 还可以在 `order by` 基础上加了指定的开创范围;
- 开窗函数需要指定聚合函数 UDAF 用于对每一行所开的窗口范围进行聚合操作.

注意: 

- 窗口函数意味着需要对每一行进行单独的聚合运算, 所以效率会变低, 谨慎使用.

# Examples

表结构: 

```sql
create city_info(
area string,
city_name string,
city_id int);
```

## 基本用法

```sql
-- 普通开窗函数
select area, city_name, city_id,
count(*) over(partition by area order by city_id desc) as rc
from city_info;

-- 开窗: 从最开始到当前行
-- 对 city_id 求和
select area, city_name, city_id,
sum(city_id) over(partition by area order by city_id desc) as rc
from city_info;

-- 开窗: 从上一行到下一行
-- 对 city_id 求和
select 
area, city_name, city_id,
sum(city_id) over(partition by area order by city_id desc rows between 1 preceding and 1 following) as rc
from city_info;

-- 没有 order by 就默认窗口为整个所在的 partition
-- 这里的 city_id 是递增因为其本来就是递增的
select 
area, city_name, city_id,
sum(city_id) over(partition by area) sum
from city_info;

-- 如果 partition by 和 order by 都不指定的话
-- 窗口为 rows between unbounded preceding and unbounded following
select 
area, city_name, city_id,
sum(city_id) over() sum
from city_info;

-- 但只要指定了 order by 那么就会自动开窗:
-- rows between unbounded preceding and current row
select 
area, city_name, city_id,
sum(city_id) over(order by city_id) sum
from city_info;

-- group by 切记是要用聚合函数的, 所以以下是错的
select 
area, city_name, city_id,
sum(city_id) over(order by city_id) sum
from city_info
group by area;
```

## rows 和 range

```sql
-- 测试 rows 和 range
-- 创建表
create table if not exists employ(
name string,
dept string,
salary int
)
row format delimited fields terminated by '\t';

-- 添加数据
insert into employ values ('张三','市场部',2000);    
insert into employ values ('赵红','技术部',2400);    
insert into employ values ('李四','市场部',3000);    
insert into employ values ('李白','技术部',3200);    
insert into employ values ('王五','市场部',4000);    
insert into employ values ('王蓝','技术部',5000);  

-- 测试
select
*,
sum(salary) over(order by salary range between 500 preceding and 500 following) ranges,
sum(salary) over(order by salary rows between 1 preceding and 1 following) rows
from employ;

-- 结果
-- 可以发现 rows 表示行; range 表示范围
name	dept	salary	ranges	rows
张三		市场部	2000	4400	4400
赵红		技术部	2400	4400	7400
李四		市场部	3000	6200	8600
李白		技术部	3200	6200	10200
王五		市场部	4000	4000	12200
王蓝		技术部	5000	5000	9000
```

## rank 排名

- `rank()` : 1 1 2

- `dense_rank()` : 1 1 3

- `row_number()` : 1 2 3

```sql

-- (RowFrame, unboundedpreceding$(), currentrow$())
-- 即函数只能开 (rows between unbounded preceding and current row)
-- 且窗口子句必须是 rows

select 
area, city_name, city_id,
rank() over(partition by area order by city_id desc rows between unbounded preceding and current row) as rc
from city_info;
```

## cume_dist 相对位置

窗口子句必须是 range

```sql
select 
area, city_name, city_id,
CUME_DIST() over(partition by area order by city_id desc range between unbounded preceding and current row) as rc
from city_info;
```

## percent_rank 相对百分比

窗口子句必须是 rows

```sql
select
area, city_name, city_id,
PERCENT_RANK() over(partition by area order by city_id desc rows between unbounded preceding and current row) ptg
from city_info;
```

## ntile 相对均匀分桶

`rows unbounded preceding and current row`

```sql
select
area, city_name, city_id,
ntile(5) over(order by city_id desc rows between unbounded preceding and current row) ptg
from city_info;
```

## lead / lag 之后 / 之前的值

取之后 / 之前 的值, 可以用于拼接, 比如页面跳转

```sql
select
area, city_name, city_id,
lead(city_id) over(partition by area order by city_id) lead
from city_info;

select
area, city_name, city_id,
lead(city_id, 2, 0) over(partition by area order by city_id) lead
from city_info;
```

# References

1. [Hive - LanguageManual Windowing And Analytics](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+WindowingAndAnalytics)

