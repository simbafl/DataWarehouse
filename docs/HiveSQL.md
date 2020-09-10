
# 案例一 通过临时表缓解计算压力

> 在将近10亿的表里执行下面逻辑，用时两个小时左右
```sql
SELECT dt as DATA_DATE,STRATEGY,group_id,SOURCE,
    count(distinct case when lower(event) not like '%push%' and event!='corner_mark_show' then udid else null end) as DAU,
    count(case when event='client_show' then 1 else null end) as TOTAL_VSHOW,
    count(distinct case when event='client_show' then vid else null end) as TOTAL_VIDEO_VSHOW,
    count(case when event='video_play' then 1 else null end) as TOTAL_VV_VP,
    count(distinct case when event='video_play' then udid else null end) as TOTAL_USERS_VP,
    count(case when event='effective_play' then 1 else null end) as TOTAL_VV_EP,
    count(distinct case when event='effective_play' then udid else null end) as TOTAL_USERS_EP,
    sum(case when event='video_over' then duration else 0 end) as TOTAL_DURATION,
    count(case when event='video_over' then 1 else null end) as TOTAL_VOVER,
    sum(case when event='video_over' then play_cnts else 0 end) as TOTAL_VOVER_PCNTS,
    count(case when event='push_video_clk' then 1 else null end) as TOTAL_PUSH_VC,
    count(distinct case when event='app_start' and body_source = 'push' then udid else null end) as TOTAL_PUSH_START,
    count(case when event='post_comment' then 1 else null end) as TOTAL_REPLY,
    count(distinct case when event='post_comment' then udid else null end) as TOTAL_USERS_REPLY
    FROM dwb.fact_log_detail
group by dt,strategy,group_id,source;
```

先不管count还是sum里面的内容，此SQL的查询时按照 `dt`,`strategy`,`group_id`,`source` 四个字段分组，然后再执行逻辑判断。

那么可以考虑在建表的时候，就按这四个字段中的N个（1或2或3或4）个字段组合分区，也就是多级分区，直接让 count(distinct xx)之类的查询定位到“更少的数据子集”，其执行效率就应该更高了，而不需要每个子任务均从全量的数据中(去重)统计。

> 查询下每个字段会有多少个分区
```sql
select count(distinct dt) as dis_dt, 
       count(distinct strategy) as dis_strategy, 
       count(distinct group_id) as dis_group_id, 
       count(distinct source) as dis_source
from dwb.fact_log_detail;
```

乘积为1000多，也就是最终会生成1000多个多级分区，还能接受。

**1. 建临时表**
```sql
create external table `dwb.fact_log_detail_tmp`(
  event string,
  udid string,
  vid string,
  duration string,
  body_source string,
  play_cnts string
)
PARTITIONED BY (
  dt string,
  source string,
  strategy string,
  group_id string
);
```

**2. 将原表数据插入新表**
```sql
insert into `dwb.fact_log_detail_tmp` partition(dt,source,strategy,group_id)
select event,udid,vid,duration,body_source,play_cnts,dt,source,strategy,group_id
from `dwb.fact_log_detail`;
```

**3. count下比对两张表的数据量是不是相同**
```sql
select count(*) from dwb.fact_log_detail_tmp;

select count(*) from dwb.fact_log_detail;
```

**4. 在新表上执行原来的逻辑**
```sql
SELECT dt as DATA_DATE,source,strategy,group_id,
    count(distinct case when lower(event) not like '%push%' and event!='corner_mark_show' then udid else null end) as DAU,
    count(case when event='client_show' then 1 else null end) as TOTAL_VSHOW,
    count(distinct case when event='client_show' then vid else null end) as TOTAL_VIDEO_VSHOW,
    count(case when event='video_play' then 1 else null end) as TOTAL_VV_VP,
    count(distinct case when event='video_play' then udid else null end) as TOTAL_USERS_VP,
    count(case when event='effective_play' then 1 else null end) as TOTAL_VV_EP,
    count(distinct case when event='effective_play' then udid else null end) as TOTAL_USERS_EP,
    sum(case when event='video_over' then duration else 0 end) as TOTAL_DURATION,
    count(case when event='video_over' then 1 else null end) as TOTAL_VOVER,
    sum(case when event='video_over' then play_cnts else 0 end) as TOTAL_VOVER_PCNTS,
    count(case when event='push_video_clk' then 1 else null end) as TOTAL_PUSH_VC,
    count(distinct case when event='app_start' and body_source = 'push' then udid else null end) as TOTAL_PUSH_START,
    count(case when event='post_comment' then 1 else null end) as TOTAL_REPLY,
    count(distinct case when event='post_comment' then udid else null end) as TOTAL_USERS_REPLY
    FROM dwb.fact_log_detail_tmp
group by dt,source,strategy,group_id;
```

最终用时不过几分钟。这种方法就是sql调优中提到的新建临时表，提前预处理数据，节省最终执行时间。

# 案例二 复杂SQL分解法