# SQL 编写与优化笔记

原则：计算数据量越少、逻辑越简单，通常越快。

### 一、SQL执行顺序

```sql
SELECT 
    city.city_name AS "City",
    COUNT(*) AS "citizen_cnt"
FROM citizen
  JOIN city ON citizen.city_id = city.city_id 
WHERE city.city_name != '上海'
GROUP BY city.city_name
HAVING COUNT(*) >= 2
-- UNION ALL
ORDER BY city.city_name ASC
LIMIT 2
```

执行顺序：

- 获取数据 (*From, Join*)
- 过滤数据 (*Where*)
- 分组 (*Group by*)
- 分组过滤 (*Having*)
- 返回查询字段 (*Select*)
- 排序与分页 (*Order by & Limit / Offset*)

如果存在UNION（如-- union all），那么UINON优先级高于order by、Limit（union 高于 limit）

当我们在执行SELECT语句时，每个步骤都会产生一张**虚拟表(virtual table）**，在执行下一步骤时，会将该虚拟表作为输入。值得注意的是，这些过程是对用户透明的。

### 二、谓词下推

#### 1 JOIN与WHERE-1

```sql
-- 数据准备
DROP TABLE IF EXISTS tb_test1;
CREATE TABLE IF NOT EXISTS tb_test1
(
    id                BIGINT
    ,name             STRING 
)
PARTITIONED BY 
(
    ds                STRING COMMENT 'YYYYMMDD'
)
;
DROP TABLE IF EXISTS tb_test2;
CREATE TABLE IF NOT EXISTS tb_test2
(
    id                BIGINT
    ,name             STRING 
)
PARTITIONED BY 
(
    ds                STRING COMMENT 'YYYYMMDD'
)
;
INSERT OVERWRITE tb_test1 PARTITION (ds = '20260227')
SELECT 1 as id,'h' as name
UNION ALL 
SELECT 2 as id,'u' as name
UNION ALL 
SELECT 3 as id,'b' as name
UNION ALL 
SELECT 4 as i,'o' as name
UNION ALL  
SELECT 5 as id,'e' as name ;

INSERT OVERWRITE tb_test2 PARTITION (ds = '20260227')
SELECT 1 as id,'u' as name
UNION ALL 
SELECT 2 as id,'b' as name ;
```

```sql
-- SQL1
SELECT t1.name,t2.name name2
FROM tb_test1 t1 
LEFT JOIN tb_test2 t2 
ON t1.id = t2.id 
AND t1.ds = t2.ds -- t1.ds = t2.ds更换为 t2.ds = '20260227' 结果一致
WHERE t1.ds = '20260227'
结果如下：
```

| name | name2 |
| ---- | ----- |
| h    | u     |
| u    | b     |
| b    | \N    |
| o    | \N    |
| e    | \N    |

```sql
-- SQL2
SELECT t1.name,t2.name name2
FROM tb_test1 t1 
LEFT JOIN tb_test2 t2 
ON t1.id = t2.id 
WHERE t1.ds = '20260227' AND t2.ds = '20260227';
结果如下：
```

| name | name2 |
| ---- | ----- |
| h    | u     |
| u    | b     |

可通过EXPLAIN查看执行计划或顺序。

逻辑解释：

关于 AND t1.ds = t2.ds 的位置：在 LEFT JOIN 中，将 t2.ds 放在 ON 子句里是非常正确的做法。如果你错误地将 t2.ds = '20260226' 放在 WHERE 子句里，LEFT JOIN 会退化为 INNER JOIN，因为 WHERE 会过滤掉 t2 为 NULL 的行。

先执行LEFT JOIN，关联条件只有 t1.id = t2.id 或者  t1.id = t2.id and t1.ds = t2.ds 结果都是如下表：

| id   | name | ds       | id1  | name2 | ds2      |
| ---- | ---- | -------- | ---- | ----- | -------- |
| 1    | h    | 20260227 | 1    | u     | 20260227 |
| 2    | u    | 20260227 | 2    | b     | 20260227 |
| 3    | b    | 20260227 | \N   | \N    |          |
| 4    | o    | 20260227 | \N   | \N    |          |
| 5    | e    | 20260227 | \N   | \N    |          |

当where 只有t1.ds = '20260227'时 结果可以保留以上5条全部结果；当where 有t1.ds = '20260227' and t2.ds = '20260227' 时，满足条件的只有前两行。

### 三、投影下推

### 四、IN、NOT IN、MAPJOIN、LEFT ANTI JOIN

1、当从表不超过1000条时，用IN或者NOT IN

2、主从表6:4、5:5、7:3 用 LEFT ANTI JOIN

3、主从表8:2 9:1 用MAPJOIN，从表不超过512M

### 五、GROUP BY与count(distinct)

上面两种方式本质就是时间与空间的权衡。
distinct需要将colA中的所有内容都加载到内存中，大致可以理解为一个hash结构，key自然就是colA的所有值。因为是hash结构，那运算速度自然就快。最后计算hash中有多少key就是最终的结果。count(distinct)只有一个reduce
在现在的海量数据环境下，需要将所有不同的值都存起来，这个内存消耗，是可想而知的。所以如果数据量特别大，可能会内存溢出。

group by的实现方式是先将colA排序。排序大家都不陌生，拿最见得快排来说，时间复杂度为O(nlogn),而空间复杂度只有O(1)。这样一来，即使数据量再大一些，group by基本也能hold住。但是因为需要做一次O (nlogn)  的排序，时间自然会稍微慢点。

总结起来就是，count(distinct)吃内存，查询快；group by空间复杂度小，在时间复杂度允许的情况下，可以发挥他的空间复杂度优势。

HIVE-去重

[DISTINCT 与 GROUP BY](https://cloud.tencent.com/developer/article/1701456)

### 六、数据倾斜-SQL优化

什么是数据倾斜

怎样发现数据倾斜

如何解决数据倾斜

其中一个：分桶：https://zhuanlan.zhihu.com/p/517105214

分桶数量和reduce数量保持一样，我们发现其实桶的概念就是MapReduce的分区的概念，两者完全相同。物理上每个桶就是目录里的一个文件，一个作业产生的桶（输出文件）数量和reduce任务个数相同，避免倾斜。

另一个：对某个字段进行hash或者取余生成一个新字段，然后水平切分大表：

大表A，insert overwrite B select A.id, A.id%100 as hash_id from A; sleect * from B where hash_id = 1 分模块取数进行计算，最后union all，避免数据倾斜。

举例说明：关联门店维度表

增量抽取：新增的用自增ID比较，发生更新的需要用更新时间和自增ID一起做判断

https://www.kanwangapp.com/thread-1197254-1-1.html

全量：表存在缓慢变化，例如订单先是创建时间、然后有签到时间，然后是拍摄完成时间，一条记录不断更新

### 七、数据重排序

order by，全局排序，将数据全部放到一个Reducer排序，对计算性能的开销较大，而且集中在单个Reduce上排序并不能起到分布式处理的效果，效率低；

通过`DISTRIBUTE BY+SORT BY`和`CLUSTER BY`两种方式来进行数据重排。

- ORDER BY：全局排序，将数据全部放到一个Reducer排序，详情请参见[ORDER BY全局排序（order_condition）](https://help.aliyun.com/zh/maxcompute/user-guide/select-syntax#section-49m-ve3-2su)。
- DISTRIBUTE BY：控制Map结果的分发，它会将具有相同字段的Map输出分发到同一个Reducer上做处理，详情请参见[DISTRIBUTE BY哈希分片（distribute_condition）](https://help.aliyun.com/zh/maxcompute/user-guide/select-syntax#section-0uo-gbq-815)。
- SORT BY：局部排序，其在数据进入Reducer前完成排序，只能保证一个Reducer内的数据是按照指定字段排序，详情请参见[SORT BY局部排序（sort_condition）](https://help.aliyun.com/zh/maxcompute/user-guide/select-syntax#section-elb-wx2-ggb)。
- CLUSTER BY : 可以理解为DISTRIBUTE BY和SORT BY某个特定字段，而且只能升序排列。

注意事项：

- 进行数据重排会占用和消耗计算资源，如果您的实例为按量付费规格，将产生计算费用，请您根据实际情况评估是否通过数据重排方式进行存储优化。
- 使用`DISTRIBUTE BY+SORT BY`、`CLUSTER BY`排序方法进行数据排序前请查看表中数值的分布情况，对列值占比大的列进行重排，存储优化效果会比较好。
- 表数据越大，进行数据重排后，存储优化效果会更好。

举例：

使用`DISTRIBUTE BY+SORT BY`方式进行数据重排：

```sql
select * from bigdata_public_dataset.tpcds_1t.customer 
distribute by c_email_address 
sort by c_email_address;
```

使用`CLUSTER BY`方式进行重排：

```sql
select * from bigdata_public_dataset.tpcds_1t.customer 
cluster by c_email_address;
```

### 八、存储优化

[合并小文件](https://help.aliyun.com/zh/maxcompute/use-cases/merge-small-files?spm=a2c4g.11186623.help-menu-27797.d_3_0_10_0.5ac878b8zopQZJ)

[数据重排](https://help.aliyun.com/zh/maxcompute/use-cases/data-reordering?spm=a2c4g.11186623.help-menu-27797.d_3_0_10_1.64561f720s56v5) 同 五 数据重拍

### 九、explain-查看执行计划

[执行计划](https://www.alibabacloud.com/help/zh/maxcompute/user-guide/explain?spm=a2c63.p38356.0.i44)

### 十、SQL优化

[SQL优化](https://www.alibabacloud.com/help/zh/maxcompute/use-cases/optimize-sql-statements?spm=a2c63.p38356.0.i53#section-mcs-xnh-0i3)

### 十一、存储优化

大表A 小表B

A 做主表：A LEFT JOIN B 或者 B RIGHT JOIN A 或者 A INNER JOIN B，使用 /*+ mapjoin(B) */ 

B做主表：B LEFT JOIN A 或者 A RIGHT JOIN B 或者 A INNER B，使用 /*+dynamicfilter(B, A) */

[动态过滤器](https://help.aliyun.com/zh/maxcompute/user-guide/dynamic-filtering?spm=a2c4g.11186623.help-menu-27797.d_2_1_2_2_11.47423b9238DAtg&scm=20140722.H_210980._.OR_help-T_cn~zh-V_1#section-bu5-bjz-tis)
