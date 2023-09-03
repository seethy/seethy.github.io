---
title: SparkSQL数据倾斜性能优化
date: 2023-09-02 10:49:59
tags: 
- sparksql
- 数据倾斜
- 性能优化
categories:
- ["组件服务", "sparksql"]
---
**背景：**
国内某大牌运营商话单数据量比较大，大概一天2T左右话单，标准话单一个小时需要处理9KW左右的话单量，标准话单处理任务在执行30分钟左右，报“too large frame”或者"Size exceeds Integer.MAX_VALUE"错误，无法正常生成标准话单数据。
**1小时任务处理数据量：**  
OTT话单 ：9KW
8303话单：无
内容数据：1千万
节目数据：无
用户数据：7百万
栏目：3
频道数据：无
节目单，tvod: 无
产品信息：100
**根因分析：**
spark任务报这2个错，主要是发生在shuffle阶段，因为Spark对每个partition所能包含的数据大小有写死的限制（约为2G），当某个partition包含超过此限制的数据时，就会抛出这类异常。
造成此异常的主要原因有:

  1. 源数据太多，partition分区数太少，导致分配到每个partition上的数据量过多，超过阈值。
  2. 数据倾斜，某列的数据分布不均衡，当某个shuffle操作是根据此列数据进行shuffle时，就会造成整个数据集发生倾斜，即某几个partition包含了大量数据，并且其数据大小超过了Spark的限制，而其他partition只包含很少的数据。

**解决方案：**
  1. 通过调整spark.sql.shuffle.partitions，增加分区数。
  2. 消除数据倾斜。
spark join主要有以下两种方式：
a) Broadcast Hash Join ：当其中一个数据集足够小时，采用Broadcast Hash Join，较小的数据集会被广播到所有Spark的executor上，并转化为一个Hash Table，之后较大数据集的各个分区会在各executor上与Hash Table进行本地的Join，各分区Join的结果合并为最终结果。
Broadcast Hash Join 没有Shuffle阶段、效率最高。但为了保证可靠性，executor必须有足够的内存能放得下被广播的数据集，所以当进两个数据集的大小都超过一个可配置的阈值之后，Spark不会采用这种Join。控制这个阈值的参数为spark.sql.autoBroadcastJoinThreshold, 中默认值为10M。
b) Sort Merge Join:  将key相同的记录重分配同一个executor上，不同的是，在每个executor上，不再构造哈希表，而是对两个分区进行排序，然后用两个下标同时遍历两个分区，如果两个下标指向的记录key相同，则输出这两条记录，否则移动key较小的下标。
Sort Merge Join也有Shuffle阶段，因此效率同样不如Broadcast Hash Join。在内存使用方面，因为不需要构造哈希表，需要的内存比Hash Join要少。
所以数据倾斜一般发生在sort merge join过程，大表跟大表关联一般建议使用sort merge join，大表的数据倾斜，可以采用将倾斜键*随机数(比如100以内的随机数), 另外一个表对应的键*100这种以空间换效率的方式； 大表跟小表关联，一般建议将小表cache, 然后通过broadcast的方式分发到各executor中提高处理性能，而且也可以避免数据倾斜的情况。

**排查和优化过程：**
因为话单原始数据量比较大，一开始怀疑默认分区数200不够，调大到1000，spark.sql.shuffle.partitions=1000，问题没有解决。
检查报错的spark sql语句，主要集中在 OTT话单同时左关联用户数据（通过usercode字段字段关联），内容数据（通过contentcode字段关联），栏目数据（通过columncode字段关联），产品信息（通过productcode字段关联）这4个数据表，通过分析spark UI 的任务执行情况，确定应该是发生数据倾斜。
检查用户表里的usercode分布，发现10%话单中usercode为空，在ETL过滤掉usercode为空的话单后，问题没有解决； 非空的usercode基本上分布比较均匀。
检查内容数据的分布，发现top5的内容数据量都在1kw 左右，数据倾斜比较严重，但是小时话单经过排查后的内容数总共就12w左右，所以修改代码在关联之前将内容先过滤下，然后再cache table, 充分利用broadcast hash join的优势，不shuffle。将spark.sql.autoBroadcastJoinThreshold配置成500M, 但是经过调整后，问题还是没有解决，还是报同样的错误。凌晨数据量比较小的情况下，偶尔任务能够成功，但是发现生成的结果数据文件差异很大，大文件甚至能够达到5G左右。
通过对sql语句进行explain分析，发现左关联的内容表，栏目表等没有通broadcast hash join还是sort merge join， 怀疑创建的内容表，栏目表等临时表 cache方式有问题。
针对spark sql broadcast的条件, 于是做了以下的测试：
```sql
create TEMPORARY view view1 asselect d.columncode,c.contentcodefrom vinsight_common_vd_vas_all_program_spark cleft join vinsight_common_vd_vas_all_column_spark don c.columncode=d.columncode;cache table cache1 as select d.columncode,c.contentcodefrom vinsight_common_vd_vas_all_program_spark cleft join vinsight_common_vd_vas_all_column_spark don c.columncode=d.columncode;create table tab1 asselect d.columncode,c.contentcodefrom vinsight_common_vd_vas_all_program_spark cleft join vinsight_common_vd_vas_all_column_spark don c.columncode=d.columncode;explain select/*+ MAPJOIN(b) */ a.usercode, b.contentcodefrom vinsight_common_md_fact_standardcdr_spark a
left join tab1 bon a.contentcode = b.contentcodewhere a.p_date>'2021-12-01';+cache1,  view1| == Physical Plan ==*Project [usercode#19861, contentcode#19683]+- *BroadcastHashJoin [contentcode#19870], [contentcode#19683], LeftOuter, BuildRight
   :- HiveTableScan [usercode#19861, contentcode#19870], HiveTableRelation `zxvmax`.`vinsight_common_md_fact_standardcdr_spark`, org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, [cdrtype#19860, usercode#19861, profilecode#19862, begintime#19863, endtime#19864, timelen#19865L, relativeurl#19866, servicetype#19867, columncode#19868, columnname#19869, contentcode#19870, contentname#19871, physicalcontentid#19872, channelcode#19873, channelname#19874, prevuecode#19875, prevuename#19876, programcode#19877, programname#19878, seriesheadcode#19879, seriesheadname#19880, cpcode#19881, cpname#19882, telecomcode#19883, ... 38 more fields], [p_provincecode#19922, p_date#19923, p_hour#19924], [isnotnull(p_date#19923), (p_date#19923 > 2021-12-01)]
   +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, string, true]))
      +- InMemoryTableScan [contentcode#19683]
            +- InMemoryRelation [columncode#19730, contentcode#19683], true, 10000, StorageLevel(disk, memory, deserialized, 1 replicas), `cache1`
                  +- *Project [columncode#19730, contentcode#19683]
                     +- SortMergeJoin [columncode#19704], [columncode#19730], LeftOuter
                        :- *Sort [columncode#19704 ASC NULLS FIRST], false, 0
                        :  +- Exchange hashpartitioning(columncode#19704, 200)
                        :     +- HiveTableScan [contentcode#19683, columncode#19704], HiveTableRelation `zxvmax`.`vinsight_common_vd_vas_all_program_spark`, org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, [opflag#19676, optime#19677, programcode#19678, bocode#19679, langtype#19680, programname#19681, programtype#19682, contentcode#19683, seriesprogramcode#19684, programhead#19685, mediaservices#19686, ratingid#19687, onlinetime#19688, offlinetime#19689, createtime#19690, countryname#19691, telecomcode#19692, mediacode#19693, seriesnum#19694, posterfilelist#19695, wggenre#19696, wgtags#19697, wgkeywords#19698, description#19699, ... 27 more fields], [p_provincecode#19727]
                        +- *Sort [columncode#19730 ASC NULLS FIRST], false, 0
                           +- Exchange hashpartitioning(columncode#19730, 200)
                              +- HiveTableScan [columncode#19730], HiveTableRelation `zxvmax`.`vinsight_common_vd_vas_all_column_spark`, org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, [opflag#19728, optime#19729, columncode#19730, bocode#19731, langtype#19732, subjectname#19733, parentcode#19734, subjecttype#19735, description#19736, mediacode#19737, telecomcode#19738, advertiseflag#19739, posterfilelist#19740, columnlock#19741, subexist#19742, updatetime#19743, p_day#19744], [p_provincecode#19745]  |
							  tab1-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--+| == Physical Plan ==*Project [usercode#20108, contentcode#20173]+- *BroadcastHashJoin [contentcode#20117], [contentcode#20173], LeftOuter, BuildRight
   :- HiveTableScan [usercode#20108, contentcode#20117], HiveTableRelation `zxvmax`.`vinsight_common_md_fact_standardcdr_spark`, org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, [cdrtype#20107, usercode#20108, profilecode#20109, begintime#20110, endtime#20111, timelen#20112L, relativeurl#20113, servicetype#20114, columncode#20115, columnname#20116, contentcode#20117, contentname#20118, physicalcontentid#20119, channelcode#20120, channelname#20121, prevuecode#20122, prevuename#20123, programcode#20124, programname#20125, seriesheadcode#20126, seriesheadname#20127, cpcode#20128, cpname#20129, telecomcode#20130, ... 38 more fields], [p_provincecode#20169, p_date#20170, p_hour#20171], [isnotnull(p_date#20170), (p_date#20170 > 2021-12-01)]
   +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, string, true]))
      +- HiveTableScan [contentcode#20173], HiveTableRelation `zxvmax`.`tab1`, org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe, [columncode#20172, contentcode#20173]  |
```
**总结:**
cache1,view1 这2种方式生成的执行计划是一样的，但是这种临时表在跟大表左关联时，还是会从原来的临时表做分解，导致还是存在SortMergeJoin, shuffle过程的。
采用创建新表这种临时表方式，然后使用hint  MAPJOIN明确使用boradcast方式可以广播小表。
最终修改sparksql为第二种方式后，任务运行结果文件比较均衡，基本上每个分区文件在50M以下，任务时间也控制在15分钟以内完成。
