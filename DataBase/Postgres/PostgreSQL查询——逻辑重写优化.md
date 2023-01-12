# PostgreSQL查询——逻辑重写优化

> ​	PostgreSQL数据库查询的入口是standard_planner的planner函数，planner函数检查当前数据库是否自定义了某些优化查询的防范。如果planner_hooker==NULL，就会调用standard_planner。下图中主要是比较重要的函数。
>
> ​	grouping_planner函数之前的属于逻辑重写优化的部分，另一部分数据物理优化。

![standard_planner](Pics/Rewrite/standard_planner.svg)

## 通用表达式（CTE）

### CTE通用表达式和子查询的区别

- CTE

​		当需要书写一些结构比较复杂，用的表也很多的SQL时，可以使用`WITH AS`语法，这个`WITH AS`是子查询的部分，**并不是真正的临时表，不会落盘，而是将数据存在内存中**。定义一个SQL片段，该片段在整个SQL中都可以使用，可以提高代码的可读性并减少重复读，减少性能成本，但是只能被当前的查询反复使用。

```sql
with 
data1 as (select * from course) ,
data2 as (select * from scoure)

select 
	data1.cno as data1_cno ,
	data2.sno as data2_sno 
from 
	data1 ,data2;
```

- SubQuery

​		子查询一般是包含了select的嵌套语句，在理解子查询的时候，需要由内向外一层层的理解。

```sql
select 
	ord.*, acc.city 
from
( 
    select 
    	accountid, 
    	orderid, 
    	ordervalue 
    from 
    	orderhistory 
    where 
    	ordervalue > '30’
) ord 
join     
    account acc 
on 
    ord.accountid = acc.accountid 

where 
    acc.country = 'China'
```



## 子查询提升

​		在数据库设计的早期，一些嵌套的SQL查询的执行采用的是嵌套的方式执行。即父查询中的每一行都查询一次子查询。这就带来了执行的低效。

​		一般来说子查询的写法要比两个表做join查询的用法更好理解一些，因为子查询的每一层都是一个独立的SQL查询。

- **相关子查询**：指的是子查询语句中引用了外层表的列属性，这就导致了外层表每获得一个元组，子查询就需要重新执行一次。
- **非相关子查询**：指子查询语句是独立的，和外层的表没有直接的关联，子查询可以单独执行一次，外层表可以重复利用子循环的执行结果。

PostgreSQL还将其分成了两类：**`子查询`**,**`子连接`**。区分的方式在于SQL语句所在位置:

如果是以范围表的方式存在的，就成为子查询

```sql
postgres=# explain select * from student ,(select * from scoure) as sc;
                              QUERY PLAN                               
-----------------------------------------------------------------------
 Nested Loop  (cost=0.00..18154.28 rows=1448400 width=98)
   ->  Seq Scan on scoure  (cost=0.00..30.40 rows=2040 width=12)
   ->  Materialize  (cost=0.00..20.65 rows=710 width=86)
         ->  Seq Scan on student  (cost=0.00..17.10 rows=710 width=86)
(4 rows)
```

如果是以表达式的方式存在的，就是子连接

```sql
postgres=# explain select (select avg(degree) from scoure),sname from student ;
                               QUERY PLAN                               
------------------------------------------------------------------------
 Seq Scan on student  (cost=35.51..52.61 rows=710 width=114)
   InitPlan 1 (returns $0)
     ->  Aggregate  (cost=35.50..35.51 rows=1 width=32)
           ->  Seq Scan on scoure  (cost=0.00..30.40 rows=2040 width=4)
(4 rows)
```

如果sql查询出现在from后面，那就是子查询，如果sql语句出现在where或者on等约束条件，亦或是投影中，也就是子连接。

### 提升子连接

​	先看下面例子：postgresql无论经过了多少次的嵌套，查询的结果都是一样的，而没有像一开始的数据库一样随着子查询的嵌套而不停地对子查询进行访问。这就涉及到一个很常见的思路：将子查询从嵌套中，提升出来。

```sql
--分页执行计划-不嵌套
db_jcxxzypt=# explain select * from db_jcxx.t_jcxxzy_tjaj order by d_slrq limit 15 offset 0;
                                                QUERY PLAN                                                 
-------------------------------------------------------------------------
 Limit  (cost=0.44..28.17 rows=15 width=879)
   ->  Index Scan using idx_ttjaj_dslrq on t_jcxxzy_tjaj  (cost=0.44..32374439.85 rows=17507700 width=879)
(2 rows)
--子查询执行计划-嵌套一层
db_jcxxzypt=# explain 
db_jcxxzypt-# select * from (
db_jcxxzypt(# select * from db_jcxx.t_jcxxzy_tjaj order by d_slrq
db_jcxxzypt(# )tab1 limit 15 offset 0;
                                                QUERY PLAN                                                 
-------------------------------------------------------------------------
 Limit  (cost=0.44..28.32 rows=15 width=879)
   ->  Index Scan using idx_ttjaj_dslrq on t_jcxxzy_tjaj  (cost=0.44..32374439.85 rows=17507700 width=879)
(2 rows)

--子查询执行计划-嵌套两层
db_jcxxzypt=# explain 
db_jcxxzypt-# select * from (
db_jcxxzypt(# select * from (
db_jcxxzypt(# select * from db_jcxx.t_jcxxzy_tjaj order by d_slrq
db_jcxxzypt(# )tab1 )tab2 limit 15 offset 0;
                                                QUERY PLAN                                                 
-------------------------------------------------------------------------
 Limit  (cost=0.44..28.32 rows=15 width=879)
   ->  Index Scan using idx_ttjaj_dslrq on t_jcxxzy_tjaj  (cost=0.44..32374439.85 rows=17507700 width=879)
(2 rows)
```

​	子连接一般出现在WHERE/ON等约束条件中，时常有SOME/ANY/ALL/IN/EXISTS等谓词同时出现。PostgreSQL主要是对ANY_SUBLINK和EXISTS_SUBLINK两种类型的子连接尝试进行提升。**EXISTS类型的相关子查询会被提升，非相关子查询连接会形成执行计划单独求解。**

#### pull_up_sublinks

​	pull_up_sublinks函数会分别对jointree进行分析，不但需要递归的处理fromlist中的范围，还需要递归处理joinExpr中的内容

#### pull_up_sublinks_qual_recure

​	主要是针对Query->jointree进行递归分析，Query->jointree节点分为三类：RangeTblRef、FromExpr或JoinExpr，针对这三种类型的节点，pull_up_sublinks_qual_recure做出了不同的处理。

![standard_planner](Pics/QueryTree/Query.png)

- RangeTblRef：RangeTblRef一定是Query->jointree上的叶子节点，因此是递归的结束条件，保存当前表的relid返回给上层，执行到这个分支通常有两种情况。
  - a) 情况一：查询一个单表，没有连接操作。递归结束，去查看子连接是否有提升的其他条件。
  - b) 情况二：查询有连接关系在，在对FromList->fromlist、JoinExpr->larg或者JoinExpr->rarg递归的过程中，遍历到了叶子节点RangeTblRef，这时候需要将这个RangeTblRef节点的relids返回给上一层，用于判断子查询是否能被提升，例如子连接的左操作树如果是左操作树的LHS(赋值操作的左侧)的一个列属性，则这个子连接不能提升。
- FromExpr：
  - a)：对FromExpr->fromlist中的节点作遍历递归，对每个节点递归调用pull_up_sublinks_jointree_recure函数，一直处理到叶子节点RangeTblRef才返回。
  - b)：调用pull_up_sublinks_qual_recure函数处理FromList->qual，对其中可能出现的ANY_SUBLINK或EXISTS_SUBLINK做处理。

- JoinExpr：

  - a)：分别调用pull_up_sublinks_jointree_recure函数递归处理JoinExpr->larg和JoinExpr->rarg，一直处理到叶子节点RangeTblRef才返回。另外还需要根据连接操作的类型区分子连接是否能够被提升。例如下面的查询无法被提升

    ```sql
    postgres=# explain select * from student left join course on student.sno> any(select cno from scoure);
                                   QUERY PLAN                               
    ------------------------------------------------------------------------
     Nested Loop Left Join  (cost=0.00..11526282.48 rows=252050 width=172)
       Join Filter: (SubPlan 1)
       ->  Seq Scan on student  (cost=0.00..17.10 rows=710 width=86)
       ->  Materialize  (cost=0.00..20.65 rows=710 width=86)
             ->  Seq Scan on course  (cost=0.00..17.10 rows=710 width=86)
       SubPlan 1
         ->  Materialize  (cost=0.00..40.60 rows=2040 width=4)
               ->  Seq Scan on scoure  (cost=0.00..30.40 rows=2040 width=4)
    (8 rows)
    ```
  - b)：调用pull_up_sublinks_qual_recure函数处理JoinExpr->quals，对其中可能出现的ANY_SUBLINK或者EXISTS_SUBLINK做处理，需要的是，根据连接类型的不同，pull_up_sublinks_qual_recure函数的available_rels1参数的输入值是不同的。