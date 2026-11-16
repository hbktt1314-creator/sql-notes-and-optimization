## SQL经典案例

### 1 产生连续数值1-100

```sql
SELECT  t.pos
FROM    (
            SELECT  1 AS id_start
                    ,101 AS id_end
        ) m
-- 制造长度是101的空字符，替换空字符为逗号，分裂为数组，最后炸裂为多行（带索引）
LATERAL VIEW POSEXPLODE(SPLIT(REPLACE(SPACE(id_end - id_start),'',','),',')) t AS
        pos
        ,val
-- 序号是从0开始的
WHERE   t.pos > 0
;
```

### 2 查询连续登陆7天以上的用户

```sql
SELECT  t4.user_id
        ,t4.fix_date
FROM    (
            SELECT  t3.user_id
                    ,t3.login_date
  								 -- 登录日期与顺序排序相减，相加不可以，可能会有日期跨月
                    ,t3.DATE_ADD(TO_DATE(login_date,'yyyy-mm-dd'),-rn) fix_date
            FROM    (
                        SELECT  t2.user_id
                                ,t2.login_date
              								-- 用户分组，登录日期排序
                                ,ROW_NUMBER() OVER (PARTITION BY t2.user_id ORDER BY t2.login_date ASC ) rn
                        FROM    (   -- 每天简化为一次登陆
                                    SELECT  t1.user_id
                                            ,SUBSTR(t1.login_date,1,10) login_date
                                    FROM    (
                                                SELECT  1 user_id
                                                        ,'2024-01-01 10:14:20' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-02 13:14:20' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-02 18:14:30' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-03 18:14:30' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-04 18:14:30' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-05 18:14:30' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-06 18:14:30' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-07 18:14:30' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-08 18:14:30' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-08 19:14:30' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-09 19:14:30' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-10 19:14:30' login_date
                                                UNION ALL
                                                SELECT  2 user_id
                                                        ,'2024-01-01 10:14:20' login_date
                                                UNION ALL
                                                SELECT  2 user_id
                                                        ,'2024-01-02 13:14:20' login_date
                                                UNION ALL
                                                SELECT  2 user_id
                                                        ,'2024-01-02 18:14:30' login_date
                                                UNION ALL
                                                SELECT  2 user_id
                                                        ,'2024-01-03 18:14:30' login_date
                                                UNION ALL
                                                SELECT  2 user_id
                                                        ,'2024-01-04 18:14:30' login_date
                                                UNION ALL
                                                SELECT  2 user_id
                                                        ,'2024-01-05 18:14:30' login_date
                                                UNION ALL
                                                SELECT  2 user_id
                                                        ,'2024-01-06 18:14:30' login_date
                                                UNION ALL
                                                SELECT  2 user_id
                                                        ,'2024-01-07 18:14:30' login_date
                                                UNION ALL
                                                SELECT  2 user_id
                                                        ,'2024-01-08 18:14:30' login_date
                                                UNION ALL
                                                SELECT  3 user_id
                                                        ,'2024-01-01 10:14:20' login_date
                                                UNION ALL
                                                SELECT  1 user_id
                                                        ,'2024-01-02 13:14:20' login_date
                                                UNION ALL
                                                SELECT  3 user_id
                                                        ,'2024-01-02 18:14:30' login_date
                                                UNION ALL
                                                SELECT  3 user_id
                                                        ,'2024-01-03 18:14:30' login_date
                                                UNION ALL
                                                SELECT  3 user_id
                                                        ,'2024-01-04 18:14:30' login_date
                                                UNION ALL
                                                SELECT  4 user_id
                                                        ,'2024-01-05 18:14:30' login_date
                                                UNION ALL
                                                SELECT  3 user_id
                                                        ,'2024-01-06 18:14:30' login_date
                                                UNION ALL
                                                SELECT  3 user_id
                                                        ,'2024-01-07 18:14:30' login_date
                                                UNION ALL
                                                SELECT  3 user_id
                                                        ,'2024-01-08 18:14:30' login_date
                                            ) t1
                                    GROUP BY t1.user_id
                                             ,SUBSTR(t1.login_date,1,10)
                                ) t2
                    ) t3
        ) t4
GROUP BY t4.user_id
         ,t4.fix_date
-- 7天及以上重复的即连续登录
HAVING  COUNT(1) >= 7;
```

### 3 增量同步

```sql
路径：
step0: mysql的binlog——>MC的实时任务——>datahub——>flink——>MC的表（增量），变更（更新、删除和新增）的数据进入MC的表；

step1：MC的增量表是日增量，生命周期36500，增量表昨日分区中时间发生时间在昨天的变更后的数据（_after_image_ = 'Y'）+增量表昨日分区中时间发生时间在今日飘逸10分钟的数据（同样是变更后的数据）；
step2：
全量表前日分区 排除 昨日增量数据（step1）+ step1的数据 = 昨日分区数据。
之后就没日循环执行以上逻辑即可。
step0是实时，1和2是每日调度一次。
```

### 4 最高峰同时直播人数

考察对拉链转化为日志的处理方式以及聚合函数的累积计算。

有如下数据记录直播平台主播上播、下播时间，根据该数据计算该平台最高峰同时直播人数

| user_id | Start_time          | End_time            |
| ------- | ------------------- | ------------------- |
| 1       | 2024-04-29 01:00:00 | 2024-04-29 02:01:05 |
| 2       | 2024-04-29 01:05:00 | 2024-04-29 02:03:18 |
| 3       | 2024-04-29 02:00:00 | 2024-04-29 04:03:22 |
| 4       | 2024-04-29 03:15:07 | 2024-04-29 04:33:21 |
| 5       | 2024-04-29 03:34:16 | 2024-04-29 06:10:45 |
| 6       | 2024-04-29 05:22:00 | 2024-04-29 07:01:08 |
| 7       | 2024-04-29 06:11:03 | 2024-04-29 09:26:05 |
| 3       | 2024-04-29 08:00:00 | 2024-04-29 12:34:27 |
| 1       | 2024-04-29 11:00:00 | 2024-04-29 16:03:18 |
| 8       | 2024-04-29 15:00:00 | 2024-04-29 17:01:05 |

1.首先对原始数据进行处理，生成主播上下播的日志数据，同时增加人数变化字段，主播上播为1，主播下播-1。新数据包含 user_id,action_time,change_cnt

```sql
--开播记录
select
user_id,
start_time as action_time,
1 as change_cnt
from t1_livestream_log
union all 
--下播记录
select
user_id,
end_time as action_time,
-1 as change_cnt
from t1_livestream_log
```

查询结果

| user_id | action_time         | change_cnt |
| ------- | ------------------- | ---------- |
| 1       | 2024-04-29 01:00:00 | 1          |
| 1       | 2024-04-29 02:01:05 | -1         |
| 2       | 2024-04-29 01:05:00 | 1          |
| 2       | 2024-04-29 02:03:18 | -1         |
| 3       | 2024-04-29 02:00:00 | 1          |
| 3       | 2024-04-29 04:03:22 | -1         |
| 4       | 2024-04-29 03:15:07 | 1          |
| 4       | 2024-04-29 04:33:21 | -1         |
| 5       | 2024-04-29 03:34:16 | 1          |
| 5       | 2024-04-29 06:10:45 | -1         |
| 6       | 2024-04-29 05:22:00 | 1          |
| 6       | 2024-04-29 07:01:08 | -1         |
| 7       | 2024-04-29 06:11:03 | 1          |
| 7       | 2024-04-29 09:26:05 | -1         |
| 3       | 2024-04-29 08:00:00 | 1          |
| 3       | 2024-04-29 12:34:27 | -1         |
| 1       | 2024-04-29 11:00:00 | 1          |
| 1       | 2024-04-29 16:03:18 | -1         |
| 8       | 2024-04-29 15:00:00 | 1          |
| 8       | 2024-04-29 17:01:05 | -1         |

2.对操作日志按照操作时间进行累积求和

```sql
with t as(
--开播记录
select
user_id,
start_time as action_time,
1 as change_cnt
from t1_livestream_log
union all 
--下播记录
select
user_id,
end_time as action_time,
-1 as change_cnt
from t1_livestream_log
)
select
user_id,
action_time,
change_cnt,
sum(change_cnt)over(order by action_time asc) as online_cnt
from t
```

查询结果

| user_id | action_time         | change_cnt | online_cnt |
| ------- | ------------------- | ---------- | ---------- |
| 1       | 2024-04-29 01:00:00 | 1          | 1          |
| 2       | 2024-04-29 01:05:00 | 1          | 2          |
| 3       | 2024-04-29 02:00:00 | 1          | 3          |
| 1       | 2024-04-29 02:01:05 | -1         | 2          |
| 2       | 2024-04-29 02:03:18 | -1         | 1          |
| 4       | 2024-04-29 03:15:07 | 1          | 2          |
| 5       | 2024-04-29 03:34:16 | 1          | 3          |
| 3       | 2024-04-29 04:03:22 | -1         | 2          |
| 4       | 2024-04-29 04:33:21 | -1         | 1          |
| 6       | 2024-04-29 05:22:00 | 1          | 2          |
| 5       | 2024-04-29 06:10:45 | -1         | 1          |
| 7       | 2024-04-29 06:11:03 | 1          | 2          |
| 6       | 2024-04-29 07:01:08 | -1         | 1          |
| 3       | 2024-04-29 08:00:00 | 1          | 2          |
| 7       | 2024-04-29 09:26:05 | -1         | 1          |
| 1       | 2024-04-29 11:00:00 | 1          | 2          |
| 3       | 2024-04-29 12:34:27 | -1         | 1          |
| 8       | 2024-04-29 15:00:00 | 1          | 2          |
| 1       | 2024-04-29 16:03:18 | -1         | 1          |
| 8       | 2024-04-29 17:01:05 | -1         | 0          |

3.求取累计求和中的最大值，即为当天最高峰同时直播人数 查询语句

```sql
with t as(
--开播记录
select
user_id,
start_time as action_time,
1 as change_cnt
from t1_livestream_log
union all 
--下播记录
select
user_id,
end_time as action_time,
-1 as change_cnt
from t1_livestream_log
)
select 
max(online_cnt) as max_online_cnt
from
(
select
user_id,
action_time,
change_cnt,
sum(change_cnt)over(order by action_time asc) as online_cnt
from t
) tt
```

查询结果

| max_online_cnt |
| -------------- |
| 3              |

### 5 哪个时间段最短且同时直播人数最多

```sql
with t as(
--开播记录
select
user_id,
start_time as action_time,
1 as change_cnt
from t1_livestream_log
union all 
--下播记录
select
user_id,
end_time as action_time,
-1 as change_cnt
from t1_livestream_log
)
    SELECT action_time, -- 达到最多直播人数的时间
    max_online_cnt, -- 最多直播人数
    min_action_time, -- 平台最早开始直播时间
    COALESCE(lag(action_time,1) OVER(ORDER BY action_time ASC),min_action_time) AS prev_action_time -- 达到最多直播人数的起始时间
FROM 
(select tt.user_id,
    tt.action_time,
    tt.change_cnt,
    tt.online_cnt,
    max(online_cnt) OVER() AS max_online_cnt,
    min(action_time) OVER() AS min_action_time
from
(
select
user_id,
action_time,
change_cnt,
-- 按照时间顺序累加直播人数
sum(change_cnt)over(order by action_time asc) as online_cnt
from t
) tt
) 
-- 只看何时达到最多直播人数
WHERE max_online_cnt = online_cnt;
```

### 6 每分钟最大直播人数

```sql
WITH a1 AS 
(
    SELECT      -- 开播记录
            user_id
            ,start_time AS action_time
            ,1 AS change_cnt
    FROM    t1_livestream_log
    UNION ALL --下播记录
    SELECT  user_id
            ,end_time AS action_time
            ,-1 AS change_cnt
    FROM    t1_livestream_log
)
,a2 AS 
(   -- 时 0-23
    SELECT  t.pos
            ,1 tag
    FROM    (
                SELECT  0 AS id_start
                        ,23 AS id_end
            ) m
    LATERAL VIEW POSEXPLODE(SPLIT(REPLACE(SPACE(id_end - id_start),'',','),',')) t AS
            pos
            ,val
)
,a3 AS 
(   -- 分，秒都是0-59
    SELECT  t.pos
            ,1 tag
    FROM    (
                SELECT  0 AS id_start
                        ,59 AS id_end
            ) m -- 制造长度是101的空字符，替换空字符为逗号，分裂为数组，最后炸裂为多行（带索引）
    LATERAL VIEW POSEXPLODE(SPLIT(REPLACE(SPACE(id_end - id_start),'',','),',')) t AS
            pos
            ,val
) -- 每分钟最大直播人数 按照分钟格式求MAX
SELECT  SUBSTR(time_1,1,16) time_minute
        ,MAX(online_cnt) minute_max_online_cnt
FROM    (
            -- 每秒的直播人数统计
            SELECT  time_1
                    ,user_id
                    ,change_cnt
                    -- 累计直播人数
                    ,SUM(change_cnt) OVER (ORDER BY time_1 ASC ) online_cnt
            FROM    (
                        -- 29日这一天时间铺开，方便后续统计到每分钟
                        SELECT  time_dim.time_1
                                ,a1.user_id
                                ,COALESCE(a1.change_cnt,0) AS change_cnt
                        FROM 
                        (  -- 拼凑年月日 时分秒 yyyy-MM-dd hh:mi:ss
                            SELECT  '2024-04-29 ' || IF(a2.pos < 10,'0' || a2.pos,a2.pos) || ':' || IF(a3.pos < 10,'0' || a3.pos,a3.pos) || ':' || IF(a4.pos < 10,'0' || a4.pos,a4.pos) time_1
                            FROM    a2
                            LEFT JOIN a3
                            ON      a2.tag = a3.tag
                            LEFT JOIN a3 a4
                            ON      a3.tag = a4.tag
                        ) time_dim
                        LEFT JOIN a1
                        ON      time_dim.time_1 = a1.action_time
                        ORDER BY time_dim.time_1
                    ) online_user_cnt
        ) online_user_cnt_detail
GROUP BY SUBSTR(time_1,1,16)
;
```

### 7 股票波峰波谷

有如下数据，记录每天每只股票的收盘价格，请查出每只股票的波峰和波谷的日期和价格； **波峰**：股票价格高于前一天和后一天价格时为波峰 **波谷**：股票价格低于前一天和后一天价格是为波谷

```sql
drop table if exists tmp_data1;
CREATE TABLE IF NOT EXISTS tmp_data1 (
    ts_code STRING, -- 主播ID
    trade_date STRING, -- 开始时间
    close DECIMAL(20,2) -- 结束时间
);

INSERT INTO TABLE tmp_data1(ts_code,trade_date,close) VALUES 
('000001.SZ', '20220104' ,16.66),
('000002.SZ', '20220104' ,20.49),
('000001.SZ', '20220105' ,17.15),
('000002.SZ', '20220105' ,21.17),
('000001.SZ', '20220106' ,17.12),
('000002.SZ', '20220106' ,21.05),
('000001.SZ', '20220107' ,17.2 ),
('000002.SZ', '20220107' ,21.89),
('000001.SZ', '20220110' ,17.19),
('000002.SZ', '20220110' ,22.16),
('000001.SZ', '20220111' ,17.41),
('000002.SZ', '20220111' ,22.3 ),
('000001.SZ', '20220112' ,17.0 ),
('000002.SZ', '20220112' ,22.05),
('000001.SZ', '20220113' ,16.98),
('000002.SZ', '20220113' ,21.53),
('000001.SZ', '20220114' ,16.33),
('000002.SZ', '20220114' ,20.7),
('000001.SZ', '20220117' ,16.22),
('000002.SZ', '20220117' ,20.87);

-- 通过lead 和 lag向下或者向上取一天
SELECT ts_code
    ,trade_date
    ,close
    -- 严格限制前后间隔一天
    ,CASE WHEN DATEDIFF(TO_DATE(trade_date,'yyyyMMdd'),TO_DATE(before_trade_date,'yyyyMMdd'),'dd') = 1
        AND DATEDIFF(TO_DATE(after_trade_date,'yyyyMMdd'),TO_DATE(trade_date,'yyyyMMdd'),'dd') = 1
        AND before_close < close
        AND after_close < close
    THEN '波峰'
    WHEN DATEDIFF(TO_DATE(trade_date,'yyyyMMdd'),TO_DATE(before_trade_date,'yyyyMMdd'),'dd') = 1
        AND DATEDIFF(TO_DATE(after_trade_date,'yyyyMMdd'),TO_DATE(trade_date,'yyyyMMdd'),'dd') = 1
        AND before_close > close
        AND after_close > close
    THEN '波谷'
    ELSE '其他' END AS tag
FROM 
(SELECT ts_code
    ,LAG(trade_date,1) OVER (PARTITION BY ts_code ORDER BY trade_date asc) before_trade_date
    ,LAG(close,1) OVER (PARTITION BY ts_code ORDER BY trade_date asc) before_close
    ,trade_date
    ,close
    ,LEAD(close,1) OVER (PARTITION BY ts_code ORDER BY trade_date asc) after_close
    ,LEAD(trade_date,1) OVER (PARTITION BY ts_code ORDER BY trade_date asc) after_trade_date
 FROM tmp_data1
)
ORDER BY ts_code asc,trade_date asc;
```

### 8 合并日期重叠的活动

```sql

CREATE TABLE IF NOT EXISTS tmp_data1 (
    hall_id STRING, -- 大厅ID
    start_date STRING, -- 开始时间
    end_date STRING -- 结束时间
);

INSERT INTO TABLE tmp_data1(hall_id,start_date,end_date) VALUES 
('1','2023-01-13','2023-01-20'),
('1','2023-01-14','2023-01-17'),
('1','2023-01-14','2023-01-16'),
('1','2023-01-18','2023-01-25'),
('1','2023-01-20','2023-01-26'),
('2','2022-12-09','2022-12-23'),
('2','2022-12-13','2022-12-17'),
('2','2022-12-20','2022-12-24'),
('2','2022-12-25','2022-12-30'),
('3','2022-12-01','2023-01-30');
WITH a as 
(   -- 排序
    SELECT hall_id
        ,start_date
        ,end_date
        ,ROW_NUMBER() OVER(PARTITION BY hall_id order by start_date asc) rn 
    FROM tmp_data1
),

b as 
(SELECT a1.hall_id
    ,a1.start_date start_date1
    ,a1.end_date end_date1
    ,a1.rn rn1
    ,a2.start_date start_date2
    ,a2.end_date end_date2
    ,a2.rn rn2
FROM a a1 
LEFT JOIN a a2 
ON a1.hall_id = a2.hall_id
 -- 找到相同厅重叠的活动，剔除相同的数据
AND a1.rn <> a2.rn
AND a1.start_date <= a2.start_date
AND a1.end_date >= a2.start_date
)
-- 将相同厅重叠的活动找到最早和最晚时间，即合并日期
SELECT hall_id
    ,min(start_date1) start_date1
    ,max(end_date2) end_date2
FROM b 
WHERE start_date2 is not null
GROUP BY hall_id
UNION ALL
-- 处理无重叠的厅活动
SELECT b1.hall_id
    ,b1.start_date1
    ,b1.end_date1
FROM b b1 
LEFT JOIN b b2 
-- 一个厅的活动与别的厅活动有重叠的时候会关联到
ON b1.rn1 = b2.rn2 
AND b1.hall_id = b2.hall_id
-- 该活动后没有日期更大的活动
WHERE b1.start_date2 is null 
-- 一个厅的活动与别的厅活动没有有重叠的
AND b2.rn2 is null;
```

### 9 查询最近一笔有效订单

```sql
drop table if exists tmp_data1;
CREATE TABLE IF NOT EXISTS tmp_data1 (
    ord_id STRING, -- 订单ID
    ord_time STRING, -- 下单时间
    user_id STRING, -- 用户ID
    is_valid STRING -- 是否有效
);
INSERT INTO TABLE tmp_data1(ord_id,ord_time,user_id,is_valid) VALUES 
('1','2023-12-11 12:01:03','a','1'),
('2','2023-12-11 12:02:06','a','0'),
('3','2023-12-11 12:03:15','a','0'),
('4','2023-12-11 12:04:20','a','1'),
('5','2023-12-11 12:05:03','a','1'),
('6','2023-12-11 12:01:02','b','1'),
('7','2023-12-11 12:03:03','b','0'),
('8','2023-12-11 12:04:01','b','1'),
('9','2023-12-11 12:07:03','b','1');

WITH a as
(-- 基础数据
  	SELECT ord_id
        ,ord_time
        ,user_id
        ,is_valid
    FROM tmp_data1
),
b as 
(SELECT a1.ord_id
        ,a1.ord_time
        ,a1.user_id
        ,a1.is_valid
        ,a2.ord_id ord_id2
        ,a2.ord_time ord_time2
        ,a2.user_id user_id2
        ,a2.is_valid is_valid2
FROM a a1 
LEFT JOIN a a2 
-- 同一用户，在该订单之前的有效订单
ON a1.user_id = a2.user_id
AND a1.ord_time > a2.ord_time
AND a2.is_valid = '1'
)
SELECT ord_id
        ,ord_time
        ,user_id
        ,is_valid
        ,ord_id2
        ,user_id2
 FROM b
-- 自己就是有效订单且在此之前无有效订单
WHERE ord_id2 is null 
AND is_valid = '1'
UNION ALL
SELECT ord_id
        ,ord_time
        ,user_id
        ,is_valid
        ,ord_id2
        ,user_id2
FROM  
(SELECT ord_id
        ,ord_time
        ,user_id
        ,is_valid
        ,ord_id2
        ,user_id2
 				-- 距离本单最近的有效订单
        ,ROW_NUMBER() OVER(PARTITION BY ord_id,user_id,user_id ORDER BY ord_time2 DESC) rn 
 FROM b
 -- 先过滤出之前有有效订单的
 WHERE ord_id2 is not null 
 )
 WHERE rn =1;
```

### 10 共同使用过3个及以上IP的用户对

```sql
drop table if exists tmp_data1;
CREATE TABLE IF NOT EXISTS tmp_data1 (
    user_id STRING, -- 用户ID
    ip STRING, -- ip
    time_stamp STRING -- 时间
);
INSERT INTO TABLE tmp_data1(user_id,ip,time_stamp) VALUES 
('2','223.104.41.101','2023-08-24 07:00:00'),
('4','223.104.41.122','2023-08-24 10:00:00'),
('5','223.104.41.126','2023-08-24 11:00:00'),
('4','223.104.41.126','2023-08-24 13:00:00'),
('1','223.104.41.101','2023-08-24 16:00:00'),
('3','223.104.41.101','2023-08-24 16:02:00'),
('2','223.104.41.104','2023-08-24 16:30:00'),
('1','223.104.41.121','2023-08-24 17:00:00'),
('2','223.104.41.122','2023-08-24 17:05:00'),
('3','223.104.41.103','2023-08-24 18:11:00'),
('2','223.104.41.103','2023-08-24 19:00:00'),
('1','223.104.41.104','2023-08-24 19:00:00'),
('3','223.104.41.122','2023-08-24 19:07:00'),
('1','223.104.41.122','2023-08-24 21:00:00');

WITH a as
(		-- 同一个用户可能多次使用同一个IP，因此去重
    SELECT user_id,ip
    FROM tmp_data1
    GROUP BY user_id,ip
),
b AS 
(		-- A表自关联的时候，避免出现用户1-用户2，用户2-用户1，这也是重复的数据
    SELECT a1.user_id user_id1
        ,a1.ip ip1 
        ,a2.user_id user_id2
        ,a2.ip ip2
    FROM a a1 
    LEFT JOIN a a2 
    ON a1.ip =a2.ip
    AND a1.user_id < a2.user_id
)

SELECT user_id1
    ,user_id2
    ,count(ip1) ip_cnt 
FROM 
(SELECT user_id1
    ,ip1 
    ,user_id2
    ,ip2
FROM b
-- 去掉无共用IP的用户数据
WHERE user_id2 is not null
)
GROUP BY user_id1
    ,user_id2 
--公用IP超过3个及以上
HAVING count(ip1) >=3 ;
```

### 11 累加刚好超过各省GDP40%的地市名称

```sql
现有各省地级市的gdp数据,求从高到底累加刚好超过各省GDP40%的地市名称，临界地市也需要。 例如:
浙江省的杭州24% 宁波 20% ,杭州+宁波=44% 大于40% 取出杭州、宁波
江苏省的苏州19% 南京 14% 无锡 12%，苏州+南京=33% ，苏州+南京+无锡=45%，取出 苏州、南京、无锡.
drop TABLE if EXISTS tmp_data1;
CREATE TABLE if not exists tmp_data1 (
    prov string COMMENT '省份'
    ,city string COMMENT '城市'
    ,gdp_amt decimal(10,2) comment 'GDP'
) COMMENT '各省地市GDP';
--插入数据
insert  into tmp_data1(prov,city,gdp_amt)
values
('浙江','杭州',20059),
('浙江','宁波',16452.8),
('浙江','温州',8730.6),
('浙江','绍兴',7791),
('浙江','嘉兴',7062.45),
('浙江','台州',6240.68),
('浙江','金华',6011.27),
('浙江','湖州',4015.1),
('浙江','衢州',2125.2),
('浙江','舟山',2100.8),
('浙江','丽水',1964.4),
('江苏','苏州',24653.37),
('江苏','南京',17421.4),
('江苏','无锡',15456.19),
('江苏','南通',11813.27),
('江苏','常州',10116.36),
('江苏','徐州',8900.44),
('江苏','扬州',7423.26),
('江苏','盐城',7403.87),
('江苏','泰州',6731.66),
('江苏','镇江',5264.07),
('江苏','淮安',5015.06),
('江苏','宿迁',4398.07),
('江苏','连云港',4363.61);

SELECT  prov
        ,city
        ,gdp_amt
        ,total_gdp
        ,prov_total_gdp
        ,rate
        ,lag_rate
        -- 获取有效数据，rate < 0.4 属于参与叠加的数据，rate >= 0.4 AND lag_rate < 0.4 属于临界数据
        ,IF((rate >= 0.4 AND lag_rate < 0.4) OR rate < 0.4,1,0) is_stay
FROM    (
            SELECT  prov
                    ,city
                    ,gdp_amt
                    ,total_gdp
                    ,prov_total_gdp
                    ,rate
                    -- 按照金额倒叙获取上个记录的占比
                    ,LAG(rate) OVER (PARTITION BY prov ORDER BY gdp_amt DESC ) lag_rate
            FROM    (
                        SELECT  prov
                                ,city
                                ,gdp_amt
                                ,total_gdp
                                ,prov_total_gdp
                                ,rate
                        FROM    (
                                    SELECT  prov
                                            ,city
                                            ,gdp_amt
                                            -- 按照金额从高到低累加
                                            ,SUM(gdp_amt) OVER (PARTITION BY prov ORDER BY gdp_amt DESC ) AS total_gdp
                                            -- 每个省全部金额
                                            ,SUM(gdp_amt) OVER (PARTITION BY prov ) AS prov_total_gdp
                                    FROM    tmp_data1
                                    ORDER BY prov ASC,gdp_amt DESC
                                )
                    )
        )
ORDER BY prov ASC,gdp_amt DESC
;
```

### 12 求连续段的起始位置和结束位置

```sql
有一张表t2_id记录了id，id不重复，但是会存在间断，求出连续段的起始位置和结束位置。
drop TABLE if EXISTS tmp_data1;
CREATE TABLE if not exists tmp_data1 (
    id BIGINT COMMENT '省份'
) COMMENT 'ID';
--插入数据
insert  into tmp_data1(id)
values
(1),
(2),
(3),
(5),
(6),
(8),
(10),
(12),
(13),
(14),
(15);
解决方案一：
select group_type,
       min(id) as start_pos,
       max(id) as end_pos
from (select id,
             sum(if(diff = 1, 0, 1)) over (order by id) as group_type
      from (select id,
                   id - lag(id) over (order by id) as diff
            from t2_id) t) tt
group by group_type;
解决方案二：
SELECT  id
        ,lag_id
        ,lead_id
        ,diff
        ,id_tag
        ,CASE   WHEN id_tag IN ('min','min_max') THEN id
                ELSE id
        END AS min_id
        ,CASE   WHEN id_tag IN ('min') THEN max_id
                WHEN id_tag IN ('min_max') THEN id
                ELSE id
        END AS max_id
FROM    (
            SELECT  id
                    ,lag_id
                    ,lead_id
                    ,diff
                    ,id_tag
                    ,LEAD(id,1) OVER (ORDER BY id ASC ) max_id
                    ,LEAD(id_tag,1) OVER (ORDER BY id ASC ) max_tag
            FROM    (
                        SELECT  id
                                ,lag_id
                                ,lead_id
                                ,diff
                                ,CASE   WHEN (diff = 1
                                            AND lag_id IS NULL)
                                            OR (diff = 1
                                            AND id - lag_id > 1) THEN 'min'
                                        WHEN (diff = 1
                                            AND lead_id IS NULL)
                                            OR (diff = 1
                                            AND lead_id - id > 1) THEN 'max'
                                        WHEN diff = 0
                                            AND lead_id - id > 1
                                            AND id - lag_id > 1 THEN 'min_max'
                                        ELSE '其他'
                                END AS id_tag
                        FROM    (
                                    SELECT  id
                                            ,lag_id
                                            ,lead_id
                                            ,IF(lead_id - id = 1 OR id - lag_id = 1,1,0) diff
                                    FROM    (
                                                SELECT  id
                                                        ,LAG(id,1) OVER (ORDER BY id ASC ) lag_id
                                                        ,LEAD(id,1) OVER (ORDER BY id ASC ) lead_id
                                                FROM    tmp_data1
                                            )
                                )
                    )
            WHERE   id_tag IN ('min','max','min_max')
        )
WHERE   (
            id_tag = 'min'
            AND     max_tag = 'max'
)
OR      (
            id_tag = 'min_max'
)
;
```
