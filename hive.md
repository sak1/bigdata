<a name="uka2L"></a>
## 部署环境
hadoop-2.10.0.tar.gz  <br />apache-hive-2.3.8-bin.tar.gz<br />PostgreSQL 9.2.24<br />openjdk-15.0.1_linux-x64_bin.tar.gz<br />

<a name="HmdJf"></a>
## 架构图：
![90.PNG](https://cdn.nlark.com/yuque/0/2021/png/2920396/1619683226451-126e7a3f-f998-4e0e-9de6-90d64e1b2a8a.png#clientId=u307c6cbd-2eab-4&from=ui&id=ue74e2936&margin=%5Bobject%20Object%5D&name=90.PNG&originHeight=471&originWidth=616&originalType=binary&size=37918&status=done&style=none&taskId=u7e257871-4a33-43d6-b282-5a37f25aba0)
<a name="qhVok"></a>
## JDK安装
1.解压openjdk-15.0.1_linux-x64_bin.tar.gz<br />2.设置环境变量
```
vi /etc/profile

export JAVA_HOME=/usr/local/jdk-15.0.1
export HIVE_HOME=/usr/local/hive
export PATH=$PATH:$HIVE_HOME/bin:$JAVA_HOME/bin

source /etc/profile


[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0003 local]# java --version
openjdk 15.0.1 2020-10-20
OpenJDK Runtime Environment (build 15.0.1+9-18)
OpenJDK 64-Bit Server VM (build 15.0.1+9-18, mixed mode, sharing)
```
<a name="gbD8O"></a>
## PostgreSQL 安装
```
#安装server
yum install postgresql-server

#安装完成验证postgresql-setup是否存在
[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0004 lib]# whereis postgresql-setup
postgresql-setup: /usr/bin/postgresql-setup /usr/share/man/man1/postgresql-setup.1.gz

#安装完成初始化数据库
[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0004 lib]# postgresql-setup initdb

#修改访问鉴权
vi /var/lib/pgsql/data/pg_hba.conf

增加host    all    all  0.0.0.0/0    md5
修改local   all    all   peer 改为local   all    all      trust

#启动
service postgresql start

#登录
psql -U postgres

#cli
create user xxxx1 with password 'xxxx';  //创建用户
create database xxxx2;  //创建database
alter user xxxx1 with password 'xxx';  //修改密码
```
psql命令参考：[https://www.cnblogs.com/my-blogs-for-everone/p/10226473.html](https://www.cnblogs.com/my-blogs-for-everone/p/10226473.html)
<a name="luP04"></a>
## hadoop安装
解压hadoop-2.10.0.tar.gz，修改hadoop-env.sh ($HOME/hadoop/etc/hadoop/hadoop-env.sh)
```
#export JAVA_HOME=${JAVA_HOME}
export JAVA_HOME=/usr/local/jdk-15.0.1
```
在hdfs上创建hive所需的文件夹并赋予权限
```
# 创建文件夹
hadoop fs -mkdir /tmp
hadoop fs -mkdir -p /user/hive/warehouse
# 给予组读权限
hadoop fs -chmod g+w /tmp
hadoop fs -chmod g+w /user/hive/warehouse
```


<a name="srFlW"></a>
## hive安装
解压apache-hive-2.3.8-bin.tar.gz进入conf目录
```
cp hive-env.sh.template hive-env.sh

vi hive-env.sh

# 配置自己的hadoop路径
HADOOP_HOME=/usr/local/hadoop-2.10.0/
# 指定配置文件路径
export HIVE_CONF_DIR=/usr/local/hive/conf/
# 指定jar包所在目录，默认读取当前安装目录下的lib
export HIVE_AUX_JARS_PATH=/usr/local/hive/lib/

```
修改hive的hive-site.xml对接postgresql服务<br />$HOME/hive/conf/hive-default.xml.template 重命名 hive-site.xml
```
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<!--
   <!-- 资源临时文件存放位置 -->
   <!-- hive用来存储不同阶段的map/reduce的执行计划的目录,
        同时也存储中间输出结果默认是/tmp/<user.name>/hive,我们实际一般会按组区分,
        然后组内自建一个tmp目录存储
    -->
	  <property>
	      <name>hive.exec.scratchdir</name>
	      <value>/user/hive/tmp</value>
	  </property>
    <!-- 设置hive仓库的HDFS上的位置,表的实际数据文件就在这里存放 -->
	  <property>
	      <name>hive.metastore.warehouse.dir</name>
	      <value>/user/hive/warehouse</value>
        <description>远程方式:hdfs://node1:9000/user/hive/warehouse</description>
	  </property>
	  <!-- 设置日志位置 -->
	  <property>
	      <name>hive.querylog.location</name>
	      <value>/user/hive/log</value>
	  </property>
-->

    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:postgresql://localhost:5432/test</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>org.postgresql.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>postgres</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>Deepexi@123</value>
    </property>
</configuration>
```
拷贝hive依赖的postgresql驱动jar包<br />cp postgresql-42.2.20.jar  $HOME/hive/lib/<br />
<br />

<a name="SqKNP"></a>
## 服务启动
启动hdfs<br />sbin/start-dfs.sh<br />启动hive
```
# 初始化metadata schema
hive/bin/schematool -dbType derby -initSchema

# 启动
metastore启动：$HIVE_HOME/bin/hive --service metastore -p 9083 
提供将DDL，DML等语句转换为MapReduce，提交到hdfs中。
不支持jdbc仅支持thrift协议

hiveserver启动: $HIVE_HOME/bin/hiveserver2 
hiveserver2 会启动一个hive服务端默认端口为：10000，可以通过beeline，jdbc，odbc的方式链接到hive。
hiveserver2启动的时候会先检查有没有配置hive.metastore.uris，如果没有会先启动一个metastore服务，
然后在启动hiveserver2。如果有配置hive.metastore.uris。会连接到远程的metastore服务。这种方式是最常用的。
hiveserver2主要用来支持jdbc obdc协议多客户端连接

cli调试:  $HIVE_HOME/bin/hive --service cli
通过 cli 启动的hive服务，第一步会先启动metastore服务，然后在启动一个客户端连接到metastore。
此时metastore服务端和客户端都在一台机器上，别的机器无法连接到metastore，所以也无法连接到hive。
这种方式不常用，一直只用于调试环节。
```
<a name="K7uGk"></a>
## 测试验证
<a name="YXzks"></a>
### hive数仓型数据库操作：
(hive语法参考：[https://blog.51cto.com/u_14048416/2342407#h3](https://blog.51cto.com/u_14048416/2342407#h3))
```
hive> show databases;
OK
default
Time taken: 1.889 seconds, Fetched: 1 row(s)

hive> create database ttt;
OK
Time taken: 0.098 seconds

hive> show databases;
OK
default
ttt
Time taken: 0.017 seconds, Fetched: 2 row(s)

hive> use ttt;
OK
Time taken: 0.02 seconds

hive> show tables;
OK
Time taken: 0.031 seconds

hive> create table if not exists employee (eid int,name String,salary String,destination String)
    > comment 'employee details'
    > ROW FORMAT DELIMITED
    >  FIELDS TERMINATED BY '\t'
    > LINES TERMINATED BY '\n'
    > STORED AS TEXTFILE;
OK
Time taken: 0.277 seconds

hive> show tables;
OK
employee
Time taken: 0.024 seconds, Fetched: 1 row(s)

hive> insert into employee(eid,name) values (1208,'jack');
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
Query ID = root_20210423145549_a239a01c-e972-41b8-b6d7-da60cee834c2
Total jobs = 3
Launching Job 1 out of 3
Number of reduce tasks is set to 0 since there's no reduce operator
Job running in-process (local Hadoop)
2021-04-23 14:55:51,787 Stage-1 map = 100%,  reduce = 0%
Ended Job = job_local299035497_0001
Stage-4 is selected by condition resolver.
Stage-3 is filtered out by condition resolver.
Stage-5 is filtered out by condition resolver.
Moving data to directory file:/user/hive/warehouse/ttt.db/employee/.hive-staging_hive_2021-04-23_14-55-49_288_9142210150905369664-1/-ext-10000
Loading data to table ttt.employee
MapReduce Jobs Launched:
Stage-Stage-1:  HDFS Read: 0 HDFS Write: 0 SUCCESS
Total MapReduce CPU Time Spent: 0 msec
OK
Time taken: 2.832 seconds

hive> select * from employee;
OK
1208    jack    NULL    NULL
Time taken: 0.102 seconds, Fetched: 1 row(s)

hive> create view empview as
    > select * from employee;
OK
Time taken: 0.095 seconds

hive> select * from empview;
OK
1208    jack    NULL    NULL
Time taken: 0.099 seconds, Fetched: 1 row(s)
```
<a name="r74Cw"></a>
### trino加载hive类型catalog实现数据查询验证：
```
//hive-catalog配置
[root@dev-wgl catalog]# pwd
/usr/local/trino-server-355/etc/catalog
[root@dev-wgl catalog]# cat hive.properties
connector.name=hive-hadoop2
hive.metastore.uri=thrift://10.201.0.204:9083
```
```
trino> show catalogs;
 Catalog
---------
 gl
 hive
 mysql
 system
(4 rows)

Query 20210423_091318_00002_gdmqk, FINISHED, 2 nodes
Splits: 36 total, 36 done (100.00%)
0.21 [0 rows, 0B] [0 rows/s, 0B/s]


trino:ttt> select * from employee;
Query 20210425_120750_00021_gdmqk failed: Partition location does not exist: file:/user/hive/warehouse/ttt.db/employee //此种情况是因为构建database元数据时未正确采用远程访问hdfs文件方式。通过hive命令查看database描述信息： desc database 名称


hive> show create table employee;
OK
CREATE TABLE `employee`(
  `eid` int,
  `name` string,
  `salary` string,
  `destination` string)
COMMENT 'employee details'
ROW FORMAT SERDE
  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
WITH SERDEPROPERTIES (
  'field.delim'='\t',
  'line.delim'='\n',
  'serialization.format'='\t')
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'file:/user/hive/warehouse/ttt.db/employee'
TBLPROPERTIES (
  'transient_lastDdlTime'='1619160952')


//查询表LOCATION配置必须是远程地址  比如：
LOCATION
  'hdfs://10.201.0.204:9000/user/hive/warehouse/pp.db/stu'


```
<a name="gsWqi"></a>
### 正确配置内部表进行trino查询验证：
```
//####hive命令查询内部表stu 及stu创建表元数据###
hive> show schemas;
OK
default
gl
postgresql_db
pp
ttt
Time taken: 1.9 seconds, Fetched: 5 row(s)
hive> use pp;
OK
Time taken: 0.023 seconds
hive> show tables;
OK
kk
stu
Time taken: 0.032 seconds, Fetched: 2 row(s)
hive> select * from stu;
OK
1       zhangsan
Time taken: 0.933 seconds, Fetched: 1 row(s)
hive> show create table stu;
OK
CREATE TABLE `stu`(
  `id` int,
  `name` string)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://10.201.0.204:9000/user/hive/warehouse/pp.db/stu'
TBLPROPERTIES (
  'transient_lastDdlTime'='1619426839')
Time taken: 0.078 seconds, Fetched: 13 row(s)

//####trino命令查询内部表stu数据   成功####
trino:pp> show tables;
 Table
-------
 kk
 stu
(2 rows)

Query 20210507_125338_00050_8z2jx, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.21 [2 rows, 29B] [9 rows/s, 139B/s]

trino:pp> select * from stu;
 id |   name
----+----------
  1 | zhangsan
(1 row)

Query 20210507_125343_00051_8z2jx, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
1.54 [1 rows, 11B] [0 rows/s, 7B/s]


```
<a name="G5rtj"></a>
### 注册关联表(connector=jdbc)查询数据验证：
```
//#######通过flink语法创建关联表元数据#########
CREATE TABLE pg_result ( 
  o_orderkey INTEGER, 
  orderstatus STRING,
  c_name STRING
) WITH ( 
  'connector' = 'jdbc',
  'url' = 'jdbc:postgresql://10.201.0.124:55432/sql_demo_test1',
  'username' = 'postgres',
  'password' = '123456',
  'table-name' = 'pg_result'
)
//#####通过hive命令查询创建该关联表语法#######
hive> show create table pg_result;
OK
CREATE TABLE `pg_result`(
)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://namenode:8020/user/hive/warehouse/faas_meta_1_1_default.db/pg_result'
TBLPROPERTIES (
  'flink.connector'='jdbc',
  'flink.password'='123456',
  'flink.schema.0.data-type'='INT',
  'flink.schema.0.name'='o_orderkey',
  'flink.schema.1.data-type'='VARCHAR(2147483647)',
  'flink.schema.1.name'='orderstatus',
  'flink.schema.2.data-type'='VARCHAR(2147483647)',
  'flink.schema.2.name'='c_name',
  'flink.table-name'='pg_result',
  'flink.url'='jdbc:postgresql://10.201.0.124:55432/sql_demo_test1',
  'flink.username'='postgres',
  'is_generic'='true',
  'transient_lastDdlTime'='1620382362')
Time taken: 0.086 seconds, Fetched: 24 row(s)

//####使用hive命令 show create table pg_result 获取语法在hive中创建新的一个关联表pg_result2  失败#
hive> CREATE TABLE `pg_result2`(
    > )
    > ROW FORMAT SERDE
    >   'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
    > STORED AS INPUTFORMAT
    >   'org.apache.hadoop.mapred.TextInputFormat'
    > OUTPUTFORMAT
    >   'org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat'
    > LOCATION
    >   'hdfs://namenode:8020/user/hive/warehouse/faas_meta_1_1_default.db/pg_result2'
    > TBLPROPERTIES (
    >   'flink.connector'='jdbc',
    >   'flink.password'='123456',
    >   'flink.schema.0.data-type'='INT',
    >   'flink.schema.0.name'='o_orderkey',
    >   'flink.schema.1.data-type'='VARCHAR(2147483647)',
    >   'flink.schema.1.name'='orderstatus',
    >   'flink.schema.2.data-type'='VARCHAR(2147483647)',
    >   'flink.schema.2.name'='c_name',
    >   'flink.table-name'='pg_result',
    >   'flink.url'='jdbc:postgresql://10.201.0.124:55432/sql_demo_test1',
    >   'flink.username'='postgres',
    >   'is_generic'='true',
    >   'transient_lastDdlTime'='1620382362');
NoViableAltException(347@[])
        at org.apache.hadoop.hive.ql.parse.HiveParser.columnNameTypeOrPKOrFK(HiveParser.java:33341)
        at org.apache.hadoop.hive.ql.parse.HiveParser.columnNameTypeOrPKOrFKList(HiveParser.java:29492)
        at org.apache.hadoop.hive.ql.parse.HiveParser.createTableStatement(HiveParser.java:6175)
        at org.apache.hadoop.hive.ql.parse.HiveParser.ddlStatement(HiveParser.java:3808)
        at org.apache.hadoop.hive.ql.parse.HiveParser.execStatement(HiveParser.java:2382)
        at org.apache.hadoop.hive.ql.parse.HiveParser.statement(HiveParser.java:1333)
        at org.apache.hadoop.hive.ql.parse.ParseDriver.parse(ParseDriver.java:208)
        at org.apache.hadoop.hive.ql.parse.ParseUtils.parse(ParseUtils.java:77)
        at org.apache.hadoop.hive.ql.parse.ParseUtils.parse(ParseUtils.java:70)
        at org.apache.hadoop.hive.ql.Driver.compile(Driver.java:468)
        at org.apache.hadoop.hive.ql.Driver.compileInternal(Driver.java:1317)
        at org.apache.hadoop.hive.ql.Driver.runInternal(Driver.java:1457)
        at org.apache.hadoop.hive.ql.Driver.run(Driver.java:1237)
        at org.apache.hadoop.hive.ql.Driver.run(Driver.java:1227)
        at org.apache.hadoop.hive.cli.CliDriver.processLocalCmd(CliDriver.java:233)
        at org.apache.hadoop.hive.cli.CliDriver.processCmd(CliDriver.java:184)
        at org.apache.hadoop.hive.cli.CliDriver.processLine(CliDriver.java:403)
        at org.apache.hadoop.hive.cli.CliDriver.executeDriver(CliDriver.java:821)
        at org.apache.hadoop.hive.cli.CliDriver.run(CliDriver.java:759)
        at org.apache.hadoop.hive.cli.CliDriver.main(CliDriver.java:686)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.apache.hadoop.util.RunJar.run(RunJar.java:221)
        at org.apache.hadoop.util.RunJar.main(RunJar.java:136)
FAILED: ParseException line 2:0 cannot recognize input near ')' 'ROW' 'FORMAT' in column name or primary key or foreign key
hive>

//######flink中查询关联表数据######
查询ok

//######trino中查询该关联表数据  报错######
trino> use fhive.faas_meta_1_1_default;
USE
trino:faas_meta_1_1_default> show tables;
   Table
-----------
 pg_result
(1 row)

Query 20210507_124455_00034_8z2jx, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.21 [1 rows, 40B] [4 rows/s, 191B/s]

trino:faas_meta_1_1_default> select * from pg_result;
Query 20210507_124506_00035_8z2jx failed: line 1:8: SELECT * not allowed from relation that has no columns
select * from pg_result

//####另外通过hive命令创建关联schema  成功####
hive> show create schema postgresql_db;
OK
CREATE DATABASE `postgresql_db`
LOCATION
  'hdfs://10.201.0.204:9000/user/hive/warehouse/postgresql_db.db'
WITH DBPROPERTIES (
  'CATALOG'='postgresql',
  'LOCATION'='jdbc:postgresql://10.201.0.204:5432/cardb',
  'PASSWORD'='Deepexi@123',
  'USER'='postgres')
Time taken: 0.035 seconds, Fetched: 8 row(s)
hive> use postgresql_db;
OK
Time taken: 0.022 seconds
hive> show tables;
OK                          //查询不到表，实际是有表的
Time taken: 0.033 seconds
hive> select * from student;
FAILED: SemanticException [Error 10001]: Line 1:14 Table not found 'student'
hive>
-------------------------------------------------------------------------
postgres=> \l
                                   List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |    Access privileges
-----------+----------+----------+-------------+-------------+--------------------------
 cardb     | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 redash_db | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/postgres            +
           |          |          |             |             | postgres=CTc/postgres   +
           |          |          |             |             | redash_user=CTc/postgres
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres             +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres             +
           |          |          |             |             | postgres=CTc/postgres
 test      | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =Tc/postgres            +
           |          |          |             |             | postgres=CTc/postgres   +
           |          |          |             |             | root=CTc/postgres
 testdb    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
(7 rows)
cardb=> \dt;
            List of relations
 Schema |    Name     | Type  |  Owner
--------+-------------+-------+----------
 public | m_student02 | table | postgres
 public | mytest      | table | postgres
 public | student     | table | postgres
(3 rows)




```
<a name="dqdnH"></a>
### 注册关联表(connector=filesystem)查询数据验证：
```
//#######通过flink语法创建关联表元数据#########
CREATE TABLE dev_orders (
  o_orderkey      INTEGER,
  o_custkey       INTEGER,
  o_orderstatus   STRING,
  o_totalprice    DOUBLE,
  o_currency      STRING,
  o_ordertime     TIMESTAMP(3),
  o_orderpriority STRING,
  o_clerk         STRING, 
  o_shippriority  INTEGER,
  o_comment       STRING,
  WATERMARK FOR o_ordertime AS o_ordertime - INTERVAL '5' MINUTE
) WITH (
  'connector' = 'filesystem',
  'path' = 's3://sql-demo/orders.tbl',
  'format' = 'csv',
  'csv.field-delimiter' = '|'
);
//#####通过hive命令查询创建该关联表语法#######
hive> show create table dev_orders;
OK
CREATE TABLE `dev_orders`(
)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://namenode:8020/user/hive/warehouse/faas_meta_1_57_default.db/dev_orders'
TBLPROPERTIES (
  'flink.connector'='filesystem',
  'flink.csv.field-delimiter'='|',
  'flink.format'='csv',
  'flink.path'='s3://sql-demo/orders-tomato.tbl',
  'flink.schema.0.data-type'='INT',
  'flink.schema.0.name'='o_orderkey',
  'flink.schema.1.data-type'='INT',
  'flink.schema.1.name'='o_custkey',
  'flink.schema.2.data-type'='VARCHAR(2147483647)',
  'flink.schema.2.name'='o_orderstatus',
  'flink.schema.3.data-type'='DOUBLE',
  'flink.schema.3.name'='o_totalprice',
  'flink.schema.4.data-type'='VARCHAR(2147483647)',
  'flink.schema.4.name'='o_currency',
  'flink.schema.5.data-type'='TIMESTAMP(3)',
  'flink.schema.5.name'='o_ordertime',
  'flink.schema.6.data-type'='VARCHAR(2147483647)',
  'flink.schema.6.name'='o_orderpriority',
  'flink.schema.7.data-type'='VARCHAR(2147483647)',
  'flink.schema.7.name'='o_clerk',
  'flink.schema.8.data-type'='INT',
  'flink.schema.8.name'='o_shippriority',
  'flink.schema.9.data-type'='VARCHAR(2147483647)',
  'flink.schema.9.name'='o_comment',
  'flink.schema.watermark.0.rowtime'='o_ordertime',
  'flink.schema.watermark.0.strategy.data-type'='TIMESTAMP(3)',
  'flink.schema.watermark.0.strategy.expr'='`o_ordertime` - INTERVAL \'5\' MINUTE',
  'is_generic'='true',
  'transient_lastDdlTime'='1617108015')
Time taken: 0.096 seconds, Fetched: 40 row(s)

//####使用hive命令 show create table dev_orders 获取语法在hive中创建新的一个关联表dev_orders2 #

//######flink中查询关联表数据######
查询ok

//######trino中查询该关联表数据  报错######
trino> use fhive.faas_meta_1_57_default;
USE
trino:faas_meta_1_57_default> show tables;
    Table
-------------
 dev_orders
 prod_orders
(2 rows)

Query 20210508_015703_00005_8z2jx, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.21 [2 rows, 85B] [9 rows/s, 407B/s]

trino:faas_meta_1_57_default> select * from dev_orders;
Query 20210508_015715_00006_8z2jx failed: line 1:8: SELECT * not allowed from relation that has no columns
select * from dev_orders

```


<a name="bmNDe"></a>
### 注册关联表(connector=kafka)查询数据验证：


<a name="Nh5lk"></a>
### trino对接hive-catalog对不同数据文件格式(CSV、JSON、Parquet、Carbon和ORC五种主流格式)查询数据验证：
```
8 rows selected (0.111 seconds)
0: jdbc:hive2://10.201.0.204:10000> use wowo;
No rows affected (0.058 seconds)
0: jdbc:hive2://10.201.0.204:10000> create table tb_text(id int) STORED AS TEXTFILE;
No rows affected (0.097 seconds)
0: jdbc:hive2://10.201.0.204:10000> create table tb_rc(id int) STORED AS RCFILE;
No rows affected (0.104 seconds)
0: jdbc:hive2://10.201.0.204:10000> create table tb_orc(id int) STORED AS ORC;
No rows affected (0.093 seconds)
0: jdbc:hive2://10.201.0.204:10000> create table tb_parquet(id int) STORED AS PARQUET;
No rows affected (0.103 seconds)
0: jdbc:hive2://10.201.0.204:10000> create table tb_avro(id int) STORED AS AVRO;
No rows affected (0.159 seconds)
0: jdbc:hive2://10.201.0.204:10000> create table tb_json(id int) STORED AS JSONFILE;
Error: Error while compiling statement: FAILED: SemanticException Unrecognized file format in STORED AS clause: 'JSONFILE' (state=42000,code=40000)
0: jdbc:hive2://10.201.0.204:10000> create table tb_sequence(id int) STORED AS SEQUENCEFILE;
No rows affected (0.121 seconds)
0: jdbc:hive2://10.201.0.204:10000> show tables;
+--------------+
|   tab_name   |
+--------------+
| car_info     |
| tb_avro      |
| tb_orc       |
| tb_parquet   |
| tb_rc        |
| tb_sequence  |
| tb_text      |
| tst          |
| u_user       |
| zz           |
+--------------+
10 rows selected (0.065 seconds)
0: jdbc:hive2://10.201.0.204:10000> insert into tb_avro values(100);
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (1.715 seconds)
0: jdbc:hive2://10.201.0.204:10000> insert into tb_orc values(100);
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (1.588 seconds)
0: jdbc:hive2://10.201.0.204:10000> insert into tb_parquet values(100);
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (1.6 seconds)
0: jdbc:hive2://10.201.0.204:10000> insert into tb_rc values(100);
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (1.575 seconds)
0: jdbc:hive2://10.201.0.204:10000> insert into tb_sequence values(100);
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (1.562 seconds)
0: jdbc:hive2://10.201.0.204:10000> insert into tb_text values(100);
WARNING: Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
No rows affected (1.562 seconds)
0: jdbc:hive2://10.201.0.204:10000>

```
```
trino> use hive.wowo;
USE
trino:wowo> show tables;
    Table
-------------
 car_info
 tb_avro
 tb_orc
 tb_parquet
 tb_rc
 tb_sequence
 tb_text
 tst
 u_user
 zz
(10 rows)

Query 20210519_014544_00003_mku84, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.21 [10 rows, 205B] [47 rows/s, 981B/s]

trino:wowo> select * from tb_avro;
 id
-----
 100
(1 row)

Query 20210519_014553_00004_mku84, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
0.73 [1 rows, 172B] [1 rows/s, 235B/s]

trino:wowo> select * from tb_orc;
 id
-----
 100
(1 row)

Query 20210519_014601_00005_mku84, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
0.21 [1 rows, 0B] [4 rows/s, 0B/s]

trino:wowo> select * from tb_parquet;
 id
-----
 100
(1 row)

Query 20210519_014612_00006_mku84, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
0.50 [1 rows, 257B] [2 rows/s, 515B/s]

trino:wowo> select * from tb_rc;
 id
-----
 100
(1 row)

Query 20210519_014623_00007_mku84, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
0.21 [1 rows, 74B] [4 rows/s, 356B/s]

trino:wowo> select * from tb_sequence;
 id
-----
 100
(1 row)

Query 20210519_014633_00008_mku84, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
0.21 [1 rows, 103B] [4 rows/s, 495B/s]

trino:wowo> select * from tb_text;
 id
-----
 100
(1 row)

Query 20210519_014643_00009_mku84, FINISHED, 1 node
Splits: 17 total, 17 done (100.00%)
0.21 [1 rows, 4B] [4 rows/s, 19B/s]

trino:wowo>

```
 avro 、orc、parquet  、rc 、sequence、text均支持查询<br />

<a name="HKwT7"></a>
## 测试总结思考
1、通过元数据查询数据思考？<br />通过hive 查询hive中创建的关联表数据是不支持的，   hive只支持查询创建非关联表的数据， 关联表的数据查询是通过flink进行的。   flink必须先建立对象(表)的元数据才可进行数据查询     trino的设计是只需加载源 即可查询源下对象元数据信息(我理解在内存中构建表的元数据)及数据<br />问题来了：flink ddl操作drop 删除的是本地，远程不会删除？trino drop删除会删除远程库对象吗？<br />
<br />2、trino如何实现对HMS中的关联表的数据进行查询？<br />
<br />3、HMS中的关联表注册途径通过什么方式？<br />
<br />4、trino注册HMS中的关联表元数据与flink注册HMS中的关联表元数据保持一致吗？<br />(flink通过连接hive的元数据管理自己的元数据表:<br />[https://blog.csdn.net/weixin_43943806/article/details/114668376?utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&dist_request_id=&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control](https://blog.csdn.net/weixin_43943806/article/details/114668376?utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control&dist_request_id=&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7EBlogCommendFromMachineLearnPai2%7Edefault-1.control))<br />
<br />5、hive-metastore元数据表管理：
```
                    List of relations
 Schema |           Name            |   Type   |  Owner
--------+---------------------------+----------+----------
 public | BUCKETING_COLS            | table    | postgres
 public | CDS                       | table    | postgres
 public | COLUMNS_V2                | table    | postgres
 public | DATABASE_PARAMS           | table    | postgres
 public | DBS                       | table    | postgres
 public | DB_PRIVS                  | table    | postgres
 public | DELEGATION_TOKENS         | table    | postgres
 public | FUNCS                     | table    | postgres
 public | FUNC_RU                   | table    | postgres
 public | GLOBAL_PRIVS              | table    | postgres
 public | IDXS                      | table    | postgres
 public | INDEX_PARAMS              | table    | postgres
 public | KEY_CONSTRAINTS           | table    | postgres
 public | MASTER_KEYS               | table    | postgres
 public | MASTER_KEYS_KEY_ID_seq    | sequence | postgres
 public | NOTIFICATION_LOG          | table    | postgres
 public | NOTIFICATION_SEQUENCE     | table    | postgres
 public | NUCLEUS_TABLES            | table    | postgres
 public | PARTITIONS                | table    | postgres
 public | PARTITION_EVENTS          | table    | postgres
 public | PARTITION_KEYS            | table    | postgres
 public | PARTITION_KEY_VALS        | table    | postgres
 public | PARTITION_PARAMS          | table    | postgres
 public | PART_COL_PRIVS            | table    | postgres
 public | PART_COL_STATS            | table    | postgres
 public | PART_PRIVS                | table    | postgres
 public | ROLES                     | table    | postgres
 public | ROLE_MAP                  | table    | postgres
 public | SDS                       | table    | postgres
 public | SD_PARAMS                 | table    | postgres
 public | SEQUENCE_TABLE            | table    | postgres
 public | SERDES                    | table    | postgres
 public | SERDE_PARAMS              | table    | postgres
 public | SKEWED_COL_NAMES          | table    | postgres
 public | SKEWED_COL_VALUE_LOC_MAP  | table    | postgres
 public | SKEWED_STRING_LIST        | table    | postgres
 public | SKEWED_STRING_LIST_VALUES | table    | postgres
 public | SKEWED_VALUES             | table    | postgres
 public | SORT_COLS                 | table    | postgres
 public | TABLE_PARAMS              | table    | postgres
 public | TAB_COL_STATS             | table    | postgres
 public | TBLS                      | table    | postgres
 public | TBL_COL_PRIVS             | table    | postgres
 public | TBL_PRIVS                 | table    | postgres
 public | TYPES                     | table    | postgres
 public | TYPE_FIELDS               | table    | postgres
 public | VERSION                   | table    | postgres
 public | aux_table                 | table    | postgres
 public | compaction_queue          | table    | postgres
 public | completed_compactions     | table    | postgres
 public | completed_txn_components  | table    | postgres
 public | hive_locks                | table    | postgres
 public | next_compaction_queue_id  | table    | postgres
 public | next_lock_id              | table    | postgres
 public | next_txn_id               | table    | postgres
 public | txn_components            | table    | postgres
 public | txns                      | table    | postgres
 public | write_set                 | table    | postgres

```
![12.PNG](https://cdn.nlark.com/yuque/0/2021/png/2920396/1620394371601-36e461c8-2354-4e51-9dfa-5713067a0c2e.png#clientId=u1dceba8a-dabe-4&from=ui&id=ub2dc2dc8&margin=%5Bobject%20Object%5D&name=12.PNG&originHeight=821&originWidth=1534&originalType=binary&size=77139&status=done&style=none&taskId=ue6e97fe0-f325-4523-a1a1-19c2e48630b)<br />
<br />元数据表关系图：<br />[https://blog.csdn.net/ThreeAspects/article/details/107393204](https://blog.csdn.net/ThreeAspects/article/details/107393204)<br />

<a name="wbxhB"></a>
## 关于Hive建表语句详解
(内表，外表，关联表，事务表，临时表，视图)<br />[https://www.pianshen.com/article/3179164342/](https://www.pianshen.com/article/3179164342/)<br />**内表**：不指定表的location参数，该参数通过读取HMS中site.xml配置hive.metastore.warehouse.dir参数指定路径，内表表删除，数据同时删除;  CREATE TABLE语法<br />**外表**：结合创建表的关键external 和自定义location对表的的数据外部存储指定路径，外表表删除，数据不删除;  CREATE EXTERNAL TABLE语法<br />**关联表**：HMS中建立关联外部数据库的表，通常在建表的TBLPROPERTIES中进行定义关联参数;<br />例子：<br />flink中创建关联表<br />CREATE TABLE `dev_orders`()<br />ROW FORMAT SERDE<br />  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'<br />STORED AS INPUTFORMAT<br />  'org.apache.hadoop.mapred.TextInputFormat'<br />OUTPUTFORMAT<br />  'org.apache.hadoop.hive.ql.io.IgnoreKeyTextOutputFormat'<br />LOCATION<br />  'hdfs://namenode:8020/user/hive/warehouse/faas_meta_1_57_default.db/dev_orders'<br />TBLPROPERTIES (//flink对表的数据查询通过TBLPROPERTIES定义的参数进行查询<br />  'flink.connector'='filesystem',<br />  'flink.csv.field-delimiter'='|',<br />  'flink.format'='csv',<br />  'flink.path'='s3://sql-demo/orders-tomato.tbl',<br />  'flink.schema.0.data-type'='INT',<br />  'flink.schema.0.name'='o_orderkey',<br />  'flink.schema.1.data-type'='INT',<br />  'flink.schema.1.name'='o_custkey',<br />  'flink.schema.2.data-type'='VARCHAR(2147483647)',<br />  'flink.schema.2.name'='o_orderstatus',<br />  'flink.schema.3.data-type'='DOUBLE',<br />  'flink.schema.3.name'='o_totalprice',<br />  'flink.schema.4.data-type'='VARCHAR(2147483647)',<br />  'flink.schema.4.name'='o_currency',<br />  'flink.schema.5.data-type'='TIMESTAMP(3)',<br />  'flink.schema.5.name'='o_ordertime',<br />  'flink.schema.6.data-type'='VARCHAR(2147483647)',<br />  'flink.schema.6.name'='o_orderpriority',<br />  'flink.schema.7.data-type'='VARCHAR(2147483647)',<br />  'flink.schema.7.name'='o_clerk',<br />  'flink.schema.8.data-type'='INT',<br />  'flink.schema.8.name'='o_shippriority',<br />  'flink.schema.9.data-type'='VARCHAR(2147483647)',<br />  'flink.schema.9.name'='o_comment',<br />  'flink.schema.watermark.0.rowtime'='o_ordertime',<br />  'flink.schema.watermark.0.strategy.data-type'='TIMESTAMP(3)',<br />  'flink.schema.watermark.0.strategy.expr'='`o_ordertime` - INTERVAL \'5\' MINUTE',<br />  'is_generic'='true',<br />  'transient_lastDdlTime'='1617108015')<br />​

//阿里云DLA自定义关联schema，通过关联schema加载table 并读取table数据<br />**CREATE** **SCHEMA** rds1 **WITH** DBPROPERTIES (<br />  **CATALOG** **=** 'mysql', <br />  **LOCATION** **=** 'jdbc:mysql://rm-2zer0vg58mfofake.mysql.rds.aliyuncs.com:3306/dla_test',<br />  **USER** **=** 'dla_test',<br />  PASSWORD **=** 'the-fake-password',<br />  VPC_ID **=** 'vpc-2zeij924vxd303kwifake',<br />  INSTANCE_ID **=** 'rm-2zer0vg58mfo5fake'<br />);<br />​

**事务表**：支持ACID能力的表;  CREATE TRANSACTIONAL TABLE语法   <br />**临时表**：已创建为临时表的表仅对当前会话可见。数据将存储在用户的暂存目录中，并在会话结束时删除。如果使用数据库中已存在的永久表的数据库/表名创建临时表，则在该会话中，对该表的任何引用都将解析为临时表，而不是永久表。如果不删除临时表或将其重命名为非冲突名称，用户将无法访问该会话中的原始表。临时表具有以下限制：不支持分区列。不支持创建索引；CREATE TEMPORARY TABLE语法<br />​<br />
<a name="tFibN"></a>
## 关于Hive的insert into 和 insert overwrite与数据分区
**1.数据分区**：数据库分区的主要目的是为了在特定的SQL操作中减少数据读写的总量以缩减响应时间，主要包括两种分区形式：水平分区与垂直分区。水平分区是对表进行行分区。而垂直分区是对列进行分区，一般是通过对表的垂直划分来减少目标表的宽度，常用的是水平分区。<br />**2.建立分区列子**
```
create external table if not exists tablename(
        a string,
        b string)
 partitioned by (year string,month string)
 row format delimited fields terminated by ',';
```
**3.hive三种方式对包含分区字段的表进行数据插入：**
```
1、静态插入数据：要求插入数据时指定与建表时相同的分区字段，如：
insert overwrite tablename （year='2019', month='06'） select a, b from tablename2;
2、动静混合分区插入：要求指定部分分区字段的值，如：
insert overwrite tablename （year='2019', month） select a, b from tablename2;
3、动态分区插入：只指定分区字段，不用指定值，如：
insert overwrite tablename （year, month） select a, b from tablename2;
```
**4.hive动态分区设置相关参数：**
```
Hive.exec.dynamic.partition  是否启动动态分区。false(不开启) true（开启）默认是 false

hive.exec.dynamic.partition.mode  打开动态分区后，动态分区的模式，有 strict和 nonstrict 两个值可选，strict 要求至少包含一个静态分区列，nonstrict则无此要求。各自的好处，大家自己查看哈。

hive.exec.max.dynamic.partitions 允许的最大的动态分区的个数。可以手动增加分区。默认1000

hive.exec.max.dynamic.partitions.pernode 一个 mapreduce job所允许的最大的动态分区的个数。默认是100
```
**5.数据插入之insert into 和 insert overwrite**<br />先说一下数据导入的几种方式：
```
（1）、从本地文件系统中导入数据到Hive表；
（2）、从HDFS上导入数据到Hive表；
（3）、在创建表的时候通过从别的表中查询出相应的记录并插入到所创建的表中；
（4）、从别的表中查询出相应的数据并导入到Hive表中。
```
insert into 和 insert overwrite区别：insert into 与 insert overwrite 都可以向hive表中插入数据，但是insert into直接追加到表中数据的尾部，而insert overwrite会重写数据，既先进行删除，再写入。如果存在分区的情况，insert overwrite会只重写当前分区数据。<br />​<br />
<a name="VmNrS"></a>
## 关于Hive数据导入导出的几种方式
**导入：**

1. 本地文件导入到Hive表；
```
vi /home/localfile/A.txt
[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0003 localfile]# cat A.txt
1,20,zs
2,23,lisi
3,32,wangwu
4,55,xiaoming

vi /home/localfile/B.txt
[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0003 localfile]# cat B.txt
11,20,zs,hz
12,23,lisi,hb
13,32,wangwu,bj
14,55,xiaoming,xg
```
```
hive> CREATE TABLE testA (
    >     id INT,
    >     vl1 string,
    >     vl2 string
    > )  ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE;
OK
Time taken: 0.048 seconds
hive> LOAD DATA LOCAL INPATH '/home/localfile/A.txt' INTO TABLE testA;
Loading data to table exportdb.testa
OK
Time taken: 0.25 seconds
hive> select * from testA;
OK
1       20      zs
2       23      lisi
3       32      wangwu
4       55      xiaoming
Time taken: 0.093 seconds, Fetched: 4 row(s)

```

2. Hive表导入到Hive表;
```
hive> insert into table testD select id,vl1,vl2 from testA;
...
OK
Time taken: 2.105 seconds
hive> select * from testD;
OK
1       20      zs
2       23      lisi
3       32      wangwu
4       55      xiaoming
Time taken: 0.084 seconds, Fetched: 4 row(s)
```

3. HDFS文件导入到Hive表;
```
hive>
    > CREATE TABLE testB (
    >     id INT,
    >     vl1 string,
    >     vl2 string,
    >     vl3 string
string    string(
    >     vl3 string
    > )ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE;
OK
Time taken: 0.045 seconds
```
```
[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0003 bin]# ./hadoop fs -mkdir /tmp/uploadtmp
...
[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0003 bin]# ./hdfs dfs -copyFromLocal /home/localfile/B.txt /tmp/uploadtmp/B.txt
...
[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0003 bin]# ./hadoop fs -cat /tmp/uploadtmp/B.txt
11,20,zs,hz
12,23,lisi,hb
13,32,wangwu,bj
14,55,xiaoming,xg


```
```
hive> LOAD DATA  INPATH '/tmp/uploadtmp/B.txt' INTO TABLE testB;
Loading data to table exportdb.testb
OK
Time taken: 0.249 seconds
hive> select * from testB;
OK
11      20      zs      hz
12      23      lisi    hb
13      32      wangwu  bj
14      55      xiaoming        xg
Time taken: 0.104 seconds, Fetched: 4 row(s)

```

4. 创建表的过程中从其他表导入;
```
hive> create table testE as select id,vl1,vl2,vl3 from testB;
....
OK
Time taken: 1.604 seconds
hive> select * from testE;
OK
11      20      zs      hz
12      23      lisi    hb
13      32      wangwu  bj
14      55      xiaoming        xg
Time taken: 0.081 seconds, Fetched: 4 row(s)

```

5. 通过sqoop将mysql库导入到Hive表；

**导出：**

1. Hive表导出到本地文件系统；
```
hive> INSERT OVERWRITE LOCAL DIRECTORY '/home/localfile/output' ROW FORMAT DELIMITED FIELDS TERMINATED by ',' select * from te
...
OK
Time taken: 1.309 seconds
```

2. Hive表导出到HDFS；
```
hive> INSERT OVERWRITE DIRECTORY '/tmp/output' select * from testA;
....
OK
Time taken: 1.435 seconds
```

3. 通过sqoop将Hive表导出到mysql库；

**hadoop常用命令：**
```
hdfs dfs -copyFromLocal /local/data /hdfs/data：将本地文件上传到 hdfs
上（原路径只能是一个文件）
hdfs dfs -put /tmp/ /hdfs/ ：和 copyFromLocal 区别是，put 原路径可以是文件夹等
hadoop fs -ls / ：查看根目录文件
hadoop fs -ls /tmp/data：查看/tmp/data目录
hadoop fs -cat /tmp/a.txt ：查看 a.txt，与 -text 一样
hadoop fs -mkdir dir：创建目录dir
hadoop fs -rmr dir：删除目录dir
```
<a name="usjJG"></a>
## 关于Hive内部表在HDFS中的目录结构
 1.默认情况，Hive内部表都属于缺省库default，在HDFS的目录为/user/hive/warehouse/（以下默认Hive的HDFS根目录为/user/hive）下
```
[root@data1 ~]# hdfs dfs -ls /user/hive/warehouse/
```
2.指定库<br /> 如果创建了数据库，并指定表为该库下的表，Hive会在/user/hive/warehouse/下创建一个以库名命名的目录（如上图的a.db，b.db分别代表数据库a，b），该库的表目录在库目录下。
```
[root@data1 ~]# hdfs dfs -ls /user/hive/warehouse/a.db
```
3.指定分区<br />如果为表设定了分区，则会在表目录下增加分区目录，目录名以“分区键名=分区值”的形式命名。如下图，表dept1中，设置了v1和v2两个名为a的分区。
```
[root@data1 ~]# hdfs dfs -ls /user/hive/warehouse/a.db/dept1
```
查看database文件路径：
```
hive > desc database temp
```
<a name="Xb68f"></a>
## 关于hive表TBLPROPERTIES内置key说明
TBLPROPERTIES实际上就是table properties,TBLPROPERTIES允许开发者定义一些自己的键值对信息。可以对TBLPROPERTIES进行查看和修改（部分可修改）。在TBLPROPERTIES中有一些预定义信息，比如last_modified_user和last_modified_time，其他的一些预定义信息包括：
```
TBLPROPERTIES ("comment"="table_comment")
TBLPROPERTIES ("hbase.table.name"="table_name")
TBLPROPERTIES ("immutable"="true") or ("immutable"="false") 
TBLPROPERTIES ("orc.compress"="ZLIB") or ("orc.compress"="SNAPPY") or ("orc.compress"="NONE")
TBLPROPERTIES ("transactional"="true") or ("transactional"="false")
TBLPROPERTIES ("NO_AUTO_COMPACTION"="true") or ("NO_AUTO_COMPACTION"="false"), the default is "false" 
TBLPROPERTIES ("compactor.mapreduce.map.memory.mb"="mapper_memory") 
TBLPROPERTIES ("compactorthreshold.hive.compactor.delta.num.threshold"="threshold_num") 
TBLPROPERTIES ("compactorthreshold.hive.compactor.delta.pct.threshold"="threshold_pct") 
TBLPROPERTIES ("auto.purge"="true") or ("auto.purge"="false") 
TBLPROPERTIES ("EXTERNAL"="TRUE") 
```

1. comment:可以用来定义表的描述信息
1. hbase.table.name：hive通过 storage handler（暂放）将hive与各种工具联系起来，这是是使用hive接入hbase时，设置的属性（暂放）
1. immutable：顾名思义‘不可变的’，当表的这个属性为true时，若表中无数据时可以insert数据，但是当表已经有数据时，insert操作会失败。不可变表用来防止意外更新，避免因脚本错误导致的多次更新，而没有报错。本人实际中还没用到这个属性。
1. orc.compress：这是orc存储格式表的一个属性，用来指定orc存储的压缩方式（暂放）。
1. transactional，NO_AUTO_COMPACTION，compactor.mapreduce.map.memory.mb，compactorthreshold.hive.compactor.delta.num.threshold，compactorthreshold.hive.compactor.delta.pct.threshold：这5个属性与hive的事务支持有关，先不做了解。
1. auto.purge：当设置为ture时，删除或者覆盖的数据会不经过回收站，直接被删除。配置了此属性会影响到这些操作： Drop Table, Drop Partitions, Truncate Table,Insert Overwrite.
1. EXTERNAL：通过修改此属性可以实现内部表和外部表的转化。
<a name="Asb69"></a>
## 涉及服务端口说明
```
tcp        0      0 0.0.0.0:9083            0.0.0.0:*               LISTEN      4221/java   //hive-metastore
tcp        0      0 0.0.0.0:10000           0.0.0.0:*               LISTEN      1425/java   //hiveserver2  jdbc端口10000
tcp        0      0 0.0.0.0:10002           0.0.0.0:*               LISTEN      1425/java   //hiveserver2  web页面10002

tcp        0      0 0.0.0.0:50010           0.0.0.0:*               LISTEN      12556/java   //datanode　控制端口
tcp        0      0 0.0.0.0:50075           0.0.0.0:*               LISTEN      12556/java   //datanode的HTTP服务器和端口
tcp        0      0 0.0.0.0:50020           0.0.0.0:*               LISTEN      12556/java   //datanode的RPC服务器地址和端口
tcp        0      0 127.0.0.1:45234         0.0.0.0:*               LISTEN      12556/java   //datanode

tcp        0      0 0.0.0.0:50090           0.0.0.0:*               LISTEN      12836/java   //SecondaryNameNode web管理端口
tcp        0      0 10.201.0.204:9000       0.0.0.0:*               LISTEN      11227/java   //NameNode
tcp        0      0 0.0.0.0:50070           0.0.0.0:*               LISTEN      11227/java   //NameNode

```
<a name="o1KsE"></a>
## hive-metastore架构设计探索
xx
<a name="NLMtG"></a>
## hiveserver2架构设计探索
xx
<a name="TmJuu"></a>
## FAQ:
1.hive异常Thelastpacketsentsuccessfullytotheserverwas0 milliseconds ago.<br />修改/etc目录下的my.cnf<br />  	[mysqld]<br />  	 wait_timeout=86400<br />
<br />2.dfs启动时需要输入密码问题
```

[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0003 hadoop-2.10.0]# ./sbin/start-dfs.sh
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by org.apache.hadoop.security.authentication.util.KerberosUtil (file:/usr/local/hadoop-2.10.0/share/hadoop/common/lib/hadoop-auth-2.10.0.jar) to method sun.security.krb5.Config.getInstance()
WARNING: Please consider reporting this to the maintainers of org.apache.hadoop.security.authentication.util.KerberosUtil
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
Incorrect configuration: namenode address dfs.namenode.servicerpc-address or dfs.namenode.rpc-address is not configured.
Starting namenodes on []
The authenticity of host 'localhost (::1)' can't be established.
ECDSA key fingerprint is SHA256:0yw2IyexJsqfbwl1GUy2Y1b/UHGwjnnLFJAw4qL4/HI.
ECDSA key fingerprint is MD5:16:ee:31:a6:54:a6:ad:c2:b6:1b:88:44:e2:86:10:cd.
Are you sure you want to continue connecting (yes/no)? yes
localhost: Warning: Permanently added 'localhost' (ECDSA) to the list of known hosts.
root@localhost's password:
localhost: Error: JAVA_HOME is not set and could not be found.
root@localhost's password:
root@localhost's password: localhost: Permission denied, please try again.

root@localhost's password: localhost: Permission denied, please try again.
```
**出现原因：**<br />OpenSSH协议里，ssh会把你每个你访问过计算机的公钥(public key)都记录在~/.ssh/known_hosts；当下次访问相同计算机时，OpenSSH会核对公钥。如果公钥不同，OpenSSH会发出警告，在更改ip后，信息会发生改变，所以出现这次现象。<br />**解决方法：**<br />方法一：rm -rf ~/.ssh/known_hosts++++++++++++++++++优点：干净利索缺点：把其他正确的公钥信息也删除，下次链接要全部重新经过认证<br />
<br />3.localhost: Error: JAVA_HOME is not set and could not be found.<br />修改hadoop-env.sh ($HOME/hadoop/etc/hadoop/hadoop-env.sh)
```
#export JAVA_HOME=${JAVA_HOME}

export JAVA_HOME=/usr/lib/jvm/jdk1.8.0_121
```

<br />4.hive-2.3.8与mysql不兼容问题
```
[root@dev-hwc-gy1-deepexi-2048-fastdata-all-216-229f-0003 bin]# schematool -dbType derby -initSchema
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hive/lib/log4j-slf4j-impl-2.6.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/local/hadoop-2.10.0/share/hadoop/common/lib/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Metastore connection URL:        jdbc:mysql://10.201.0.205:3306/test?createDatabaseIfNotExist=true
Metastore Connection Driver :    com.mysql.cj.jdbc.Driver
Metastore connection User:       root
Starting metastore schema initialization to 2.3.0
Initialization script hive-schema-2.3.0.derby.sql
Error: You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"APP"."NUCLEUS_ASCII" (C CHAR(1)) RETURNS INTEGER LANGUAGE JAVA PARAMETER STYLE ' at line 1 (state=42000,code=1064)
org.apache.hadoop.hive.metastore.HiveMetaException: Schema initialization FAILED! Metastore state would be inconsistent !!
Underlying cause: java.io.IOException : Schema script failed, errorcode 2
Use --verbose for detailed stacktrace.
*** schemaTool failed ***

```
解决办法：换成postgresql数据库对接<br />​

5.[执行start-dfs.sh，namenode无法启动](https://www.cnblogs.com/jichui/articles/7777832.html)<br />每次开机都得重新格式化一下namenode才可以，其实问题就出在tmp文件，默认的tmp文件每次重新开机会被清空，与此同时namenode的格式化信息就会丢失；于是我们得重新配置一个tmp文件目录，首先在home目录下建立一个hadoop_tmp目录：<br />sudo mkdir /home/hadoop_tmp<br />然后修改Hadoop/conf目录里面的core-site.xml文件，加入以下节点：<br />               <property><br />                        <name>hadoop.tmp.dir</name><br />                        <value>/home/hadoop_tmp</value><br />               </property><br />重新格式化Namenode<br />              hadoop namenode -format<br />然后启动hadoop<br />              start-all.sh<br />执行下JPS命令就可以看到NameNode了
