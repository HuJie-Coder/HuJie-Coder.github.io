---
layout: post
title: Clickhouse MergeTree表引擎
---
# MergeTree
```MergeTree```系列的表引擎威力很大，但是起作用的前提是进行了合并。正常情况下，```CH``` 在一定的时间间隔内会自动进行合并，但是在有些情况下，可以进行手动合并，手动合并使用```OPTIMIZE TABLE tableName FINAL```进行合并，这种方式开销很大，生产环境慎用! ```MergeTree```系列的表引擎有一些相同的功能，这些功能适用于该系列的所有表引擎。

- TTL(Time To Live)
    支持列级别的 TTL和表级别的 TTL，表示数据的存活时间，如果是列级别的 TTL ，则会删除这一列的数据，如果是表级别的 TTL，则会删除整张表的数据。若同时设置了列级别和表级别的 TTL ，则会以先到期的那个时间为主。

## ReplacingMergeTree
```ReplacingMergeTree```解决数据重复问题，这里的数据重复只能在一定程度上解决，而不能完全解决。```ORDER BY```后面接的是需要去重的字段，即同一组数据，```ReplacingMergeTree```去重的策略有两种，如果没有设置版本号，则保留同一组重复数据中的最后一行，如果设置了版本号，则保留同一组数据中版本号最大的一行。
```sql
-- 没有设置版本号，保留同一组重复数据中的最后一行
CREATE TABLE IF NOT EXISTS test_jayden.replacing_mt_test
(
    id          String,
    name        String,
    create_time DateTime
) ENGINE = ReplacingMergeTree()
      PARTITION BY toYYYYMMDD(create_time)
      ORDER BY (id, name)
;
-- 设置版本号，保留版本号最大的一行
CREATE TABLE IF NOT EXISTS test_jayden.replacing_mt_test_v2
(
    id      String,
    name    String,
    version UInt8
) ENGINE = ReplacingMergeTree(version)
      ORDER BY (id, name)
;
```
## SummingMergeTree
```SummingMergeTree```的作用是用来做数据的汇总，并且这种汇总条件是预先明确的。```ORDER BY```后面接一组字段，其余除分区字段之外的字段均作为汇总字段
```sql
CREATE TABLE test_jayden.summing_table
(
    id          String,
    city        String,
    v1          UInt32,
    v2          Float64,
    create_time DateTime
) engine = SummingMergeTree()
      PARTITION BY toYYYYMM(create_time)
      ORDER BY (id, city)
;
```
## AggregatingMergeTree
###  创建语句
```sql
CREATE TABLE test_jayden.agg_table
(
    id          String,
    city        String,
    code        AggregateFunction(uniq, String),
    value       AggregateFunction(sum, UInt32),
    create_time DateTime
) ENGINE = AggregatingMergeTree()
      PARTITION BY toYYYYMM(create_time)
      ORDER BY (id, city)
;
```

###  数据写入
&emsp; ```AggregatingMergeTree``` 和平常的表插入不同，需要聚合的字段要借助```*State```函数
```sql
INSERT INTO test_jayden.agg_table
SELECT 'A001', 'wuhan', 
        uniqState('code1'), 
        sumState(toUInt32(100)), '2020-01-01 15:00:00'
;
```

###  数据查询
&emsp; ```AggregatingMergeTree``` 查询的时候，需要聚合的字段要借助```*Merge```函数来进行查询
```sql
SELECT id, city, uniqMerge(code), sumMerge(value)
FROM test_jayden.agg_table
GROUP BY id, city
;
```

###  解决方案
&emsp; 对于 ```AggregatingMergeTree``` 引擎插入和查询带来的问题，可以从表的设计角度来解决，从而达到对用户透明。具体的做法是存数据的时候使用 ```MergeTree```作为底表，用于存储全量的明细数据，并以此对外提供实时查询，接着新建一张 ```AggregatingMergeTree```物理视图表，这张表提供定期聚合的作用，最后再提供一张视图表，提供对外的查询服务。下面是具体的实例
```sql
-- 底表，使用 MergeTree 作为表引擎
CREATE TABLE test_jayden.agg_table_basic
(
    id    String,
    city  String,
    code  String,
    value UInt32
) ENGINE = MergeTree()
      PARTITION BY city
      ORDER BY (id, city)
;

-- 物理视图表，起聚合作用
CREATE MATERIALIZED VIEW test_jayden.agg_materialized_view
    ENGINE = AggregatingMergeTree()
        ORDER BY (id, city)
AS
SELECT id,
       city,
       uniqState(code) AS code,
       sumState(value) AS value
FROM test_jayden.agg_table_basic
GROUP BY id, city
;

-- 视图表，提供对外查询服务
CREATE VIEW test_jayden.table_view
AS
SELECT id,
       city,
       uniqMerge(code) AS code,
       sumMerge(value) AS value
FROM test_jayden.agg_materialized_view
GROUP BY id, city
;
```

## CollapsingMergeTree 
```CollapsingMergeTree```的设计思路是以增加代替删除，所以会设置一个标记位置，这个标记位置记录这一行的状态，```1```表示这一行是有效数据，```-1```表示这是一行无效数据，表示这行数据需要被删除。```CollapsingMergeTree```在折叠数据的时候，遵循一定的规则
- ```sign=1```比```sign=-1```的数据多一行，保留```sign=1```的数据
- ```sign=1```比```sign=-1```的数据少一行，保留```sign=-1```的数据
-  ```sign=1```比```sign=-1```的数据行数一样多
    - 如果最后一行是```sign=1```，保留第一行```sign=-1```和最后一行```sign=1```的数据
    - 如果最后一行是```sign=-1```，都不保留

###  创建语句
```sql
CREATE TABLE test_jayden.collpase_table(
    id String,
    code Int32,
    create_time DateTime,
    sign Int8
)ENGINE = CollapsingMergeTree(sign)
PARTITION BY toYYYYMM(create_time)
ORDER BY id
;
```
值得注意的是，```CollapsingMergeTree```对数据的写入顺序有着严格要求，这种现象是```CollapsingMergeTree```的处理机制引起的，因为他要求```sign=1```和```sign=-1```的数据相邻。

## VersionedCollapsingMergeTree
```VersionedCollapsingMergeTree```在```CollapsingMergeTree```的基础上加入版本号，版本号会作为排序条件增加到 ```ORDER BY```末端。这种方式的好处是处理数据的时候对写入顺序没有要求，在同一个分区内，任意顺序的数据都能完成折叠操作。
```sql
CREATE TABLE test_jayden.ver_collpase_table
(
    id          String,
    code        Int32,
    create_time DateTime,
    sign        Int8,
    ver         UInt8
) ENGINE = VersionedCollapsingMergeTree(sign,ver)
      PARTITION BY toYYYYMM(create_time)
      ORDER BY id
;
```
