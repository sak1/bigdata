# Clickhouse部署与测试

1. 测试环境

​    资源环境：3* （16vcpu 32GB）

   测试数据：tpcds 500GB的数据量

  clickhouse版本：21.7.5.29

1. 参考文档

https://github.com/ClickHouse/ClickHouse/issues/9887

https://github.com/Altinity/tpc-ds

https://blog.csdn.net/alitech2017/article/details/106102796

https://www.jianshu.com/p/4a9b838a4be8[





](https://blog.csdn.net/vkingnew/article/details/107413830)

1. 下载

export LATEST_VERSION=21.7.5.29

curl -O https://repo.clickhouse.tech/tgz/clickhouse-common-static-$LATEST_VERSION.tgz

curl -O https://repo.clickhouse.tech/tgz/clickhouse-common-static-dbg-$LATEST_VERSION.tgz

curl -O https://repo.clickhouse.tech/tgz/clickhouse-server-$LATEST_VERSION.tgz

curl -O https://repo.clickhouse.tech/tgz/clickhouse-client-$LATEST_VERSION.tgz

1. 安装

解压后执行 

  bash -x clickhouse-common-static-20.2.1.2183/install/doinst.sh  

 bash -x clickhouse-common-static-dbg-20.2.1.2183/install/doinst.sh 

 bash -x clickhouse-server-20.2.1.2183/install/doinst.sh 

 bash -x clickhouse-client-20.2.1.2183/install/doinst.sh

1. 配置

16vcpu 32GB 三个

 

共配置了三个clickhouse节点，无副本

编辑/etc/clickhouse-server/config.xml 和两个配置文件.

   <listen_host>10.201.0.202</listen_host>

​    <remote_servers incl="clickhouse_remote_servers" >        <!-- Test only shard config for testing distributed storage -->

​        <test_cluster_noreplica>

​            <shard>

​                <replica>

​                    <host>10.201.0.203</host>

​                    <port>9200</port>

​                </replica>

​            </shard>

​            <shard>

​                <replica>

​                    <host>10.201.0.204</host>

​                    <port>9200</port>

​                </replica>

​            </shard>

​            <shard>

​                <replica>

​                    <host>10.201.0.202</host>

​                    <port>9200</port>

​                </replica>

​            </shard>

​        </test_cluster_noreplica>

​    </remote_servers>



​    <zookeeper incl="zookeeper-servers" optional="true" />    <zookeeper>

​    <node index="1">

​    <host>dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0001</host>

​    <port>2181</port>

​    </node>

​    <node index="2">

​    <host>dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0002</host>

​    <port>2181</port>

​    </node>

​    <node index="3">

​    <host>dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0003</host>

​    <port>2181</port>

​    </node>

​    </zookeeper>



​    <macros>      <cluster>test_cluster_noreplica</cluster>

​      <layer>01</layer>

​      <replica>dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0001</replica>

​    </macros>



users.xml 的配置文件的修改

<max_memory_usage>25000000000</max_memory_usage>

1. 安装配置zookeeper

略.

1. 启动clickhouse

systemctl restart clickhouse-server



1. 创建表

7.1 创建local表 

需要有部分数据类型的修改

create table if not exists sf500.call_center_local ON CLUSTER '{cluster}'( \      cc_call_center_sk UInt64 \

,     cc_call_center_id FixedString(16) \

,     cc_rec_start_date date \

,     cc_rec_end_date date \

,     cc_closed_date_sk UInt64 \

,     cc_open_date_sk UInt64 \

,     cc_name FixedString(50) \

,     cc_class FixedString(50) \

,     cc_employees UInt32 \

,     cc_sq_ft UInt32 \

,     cc_hours FixedString(20) \

,     cc_manager FixedString(40) \

,     cc_mkt_id UInt32 \

,     cc_mkt_class FixedString(50) \

,     cc_mkt_desc FixedString(100) \

,     cc_market_manager FixedString(40) \

,     cc_division UInt32 \

,     cc_division_name FixedString(50) \

,     cc_company UInt32 \

,     cc_company_name FixedString(50) \

,     cc_street_number FixedString(10) \

,     cc_street_name FixedString(60) \

,     cc_street_type FixedString(15) \

,     cc_suite_number FixedString(10) \

,     cc_city FixedString(60) \

,     cc_county FixedString(30) \

,     cc_state FixedString(2) \

,     cc_zip FixedString(10) \

,     cc_country FixedString(20) \

,     cc_gmt_offset decimal(5,2) \

,     cc_tax_percentage decimal(5,2) \

) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{layer}/call_center_local', '{replica}')  ORDER BY (cc_call_center_sk) SETTINGS index_granularity = 8192;

7.2 创建分布式表

create table if not exists sf500.call_center ON CLUSTER '{cluster}'( \      cc_call_center_sk UInt64 \

,     cc_call_center_id FixedString(16) \

,     cc_rec_start_date date \

,     cc_rec_end_date date \

,     cc_closed_date_sk UInt64 \

,     cc_open_date_sk UInt64 \

,     cc_name FixedString(50) \

,     cc_class FixedString(50) \

,     cc_employees UInt32 \

,     cc_sq_ft UInt32 \

,     cc_hours FixedString(20) \

,     cc_manager FixedString(40) \

,     cc_mkt_id UInt32 \

,     cc_mkt_class FixedString(50) \

,     cc_mkt_desc FixedString(100) \

,     cc_market_manager FixedString(40) \

,     cc_division UInt32 \

,     cc_division_name FixedString(50) \

,     cc_company UInt32 \

,     cc_company_name FixedString(50) \

,     cc_street_number FixedString(10) \

,     cc_street_name FixedString(60) \

,     cc_street_type FixedString(15) \

,     cc_suite_number FixedString(10) \

,     cc_city FixedString(60) \

,     cc_county FixedString(30) \

,     cc_state FixedString(2) \

,     cc_zip FixedString(10) \

,     cc_country FixedString(20) \

,     cc_gmt_offset decimal(5,2) \

,     cc_tax_percentage decimal(5,2) \

) ENGINE = Distributed('{cluster}' , sf500, call_center_local, rand());



1. 导入数据

形如： cat call_center_1_4.dat | clickhouse-client --user default -h 10.201.0.202 --port 9200 --format_csv_delimiter="|" --max_partitions_per_insert_block=100 --database="sf500" --query="INSERT INTO call_center FORMAT CSV"



1. 优化表

OPTIMIZE TABLE sf500.call_center_local ON CLUSTER '{cluster}';





二、clickhouse的优势



1.列存储 

2.向量化执行

3.编码压缩

4.多索引

5.物化视图（Cube/Rollup）

6.SQL 方言：在常用场景下，兼容 ANSI SQL，并支持 JDBC、ODBC 等丰富接口。

7.权限管控：支持 Role-Based 权限控制，与关系型数据库使用体验类似。

8.多机多核并行计算：ClickHouse 会充分利用集群中的多节点、多线程进行并行计算，提高性能。

9.近似查询：支持近似查询算法、数据抽样等近似查询方案，加速查询性能。

10.Colocated Join：数据打散规则一致的多表进行 Join 时，支持本地化的 Colocated Join，提升查询性能







三、clickhouse 与apache doris的对比



1. 网上的测试结论参考

1.1 单表查询Clickhouse快， Clickhouse支持Vectorized与SIMD。Doris支持Vectorized，Simd在收费版本才有
1.2 Join查询两者各有优劣，数据量小情况下Clickhouse好，数据量大Doris好 

2.应用场景

2.1. 用户行为分析系统

行为分析系统的表可以打成一个大的宽表形式，join 的形式相对少一点，可以实现路径分析、漏斗分析、路径转化等功能

2.2. BI报表

结合clickhouse的实时查询功能，可以实时的做一些需要及时产出的灵活BI报表需求，包括并成功应用于留存分析、用户增长、广告营销等

2.3. 监控系统

视频播放质量、CDN质量，系统服务报错信息等指标，也可以接入ClickHouse，结合Kibana实现监控大盘功能

2.4. ABtest

其高效的存储性能以及丰富的数据聚合函数成为实验效果分析的不二选择。离线和实时整合后的用户命中的实验分组对应的行为日志数据最终都导入了clickhouse，用于计算用户对应实验的一些埋点指标数据（主要包括pv、uv）。

业界可以参考：https://www.jianshu.com/p/79d31a72978f（Athena-贝壳流量实验平台设计与实践）

2.5. 特征分析

使用Clickhouse针对大数据量的数据进行聚合计算来提取特征

场景举例：用户行为实时分析OLAP应用场景



### **3.ClickHouse的不完美**

- 1.不支持事物。
- 2.不支持Update/Delete操作。

- 3.支持有限操作系统。

   . 4 Mutation 查询 ClickHouse 提供了 Delete 和 Update 的能力，这类操作被称为 Mutation 查询，它可以看做 Alter 的一种



4.架构



4.1 clickhouse 单机架构

![img](https://cdn.nlark.com/yuque/0/2021/png/21485798/1628063399043-5dd63f9f-babf-4723-b392-c996fdb340c8.png)



4.2 clickhouse 常用集群架构

4.2.1 clickhouse 依赖zk的集群架构

![img](https://cdn.nlark.com/yuque/0/2021/jpeg/21485798/1628064920480-1296f256-f899-4d52-b082-2dd3d79039bb.jpeg)



4.2.2 clickhouse 不依赖zk的集群架构

![img](https://cdn.nlark.com/yuque/0/2021/webp/21485798/1628065339005-e927f43b-8ee7-4264-b190-5faebaceb318.webp)



5.clickhouse join 查询放大问题，需要添加global关键字

分布式查询的存在查询放大问题，放大次数是 N的平方(N = 分片数量)。所以说，如果一张表有10个分片，那么一次分布式 IN 查询的背后会涉及100次查询 

https://cloud.tencent.com/developer/article/1621346 

未使用global SELECT uniq(id) FROM test_query_local WHERE repo = 100 

AND id IN (SELECT id FROM test_query_all WHERE repo = 200)

A

1.子查询 执行了2次

B

1.子查询被执行了2次



 SELECT uniq(id) FROM test_query_local WHERE repo = 100 

AND id GLOBAL IN (SELECT id FROM test_query_all WHERE repo = 200)

A

1.子查询1次

2.汇总结果

3.执行关联查询

B  

1.子查询1次

2.汇总结果

3.执行关联查询



1. 慢查询优化

ck的sql慢的优化方向分区，原则是尽量把经常一起用到的数据放到相同区（也可以根据where条件来分区），如果一个区太大再放到多个区，

主键（索引，即排序）order by字段选择： 就是把where 里面肯定有的字段加到里面，where 中一定有的字段放到第一位，注意字段的区分度适中即可 区分度太大太小都不好，因为ck的索引时稀疏索引，采用的是按照固定的粒度抽样作为实际的索引值，不是mysql的二叉树，所以不建议使用区分度特别高的字段。



7.clickhouse join的效率问题

对数据量大的几个表使用了分区，并且按照primary key进行排序 如：PARTITION BY cr_returned_date_sk PRIMARY KEY (cr_item_sk, cr_order_number)

ORDER BY (cr_item_sk, cr_order_number)

SETTINGS index_granularity = 8192；

**对内存的消耗非常的大，到trino中500GB的数据量的查询都可以运行，在clickhouse tpcds  100GB数据量，运行了4个query，3个内存溢出**

**q3.sql q37.sql q55.sql 都会内存溢出，q55.sql 10.832 sec**

[
](https://blog.csdn.net/h2604396739/article/details/86172756)clickhouse join的慢的原因是ck没有完全的实现shuffle join，只实现了Broadcast JOIN Colocate JOIN

spark对shuffle join的实现参考如下：

shuffle Hash join 设计思路剖析（大表join小表）第一步：一般情况下，streamIter为大表，buildIter为小表，不用关心哪个表为streamIter，哪个表为buildIter，这个spark会根据join语句自动帮我们完成。

第二步： 将具有相同性质的（如Hash值相同）join key 进行Shuffle到同一个分区。

第三步：先把小表广播到所有大表分区所在节点，然后根据buildIter Table的join key构建Hash Table，把每一行记录都存进HashTable

第四步：扫描streamIter Table 每一行数据，使用相同的hash函数匹配 Hash Table中的记录，匹配成功之后再检查join key 是否相等，最后join在一起