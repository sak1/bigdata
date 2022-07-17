

<a name="nnYjf"></a>
# 1 Schema演化
Iceberg schema演化支持添加，删除，更新或重命名，并且没有副作用。
<a name="gSg9O"></a>
## 1.1 重命名iceberg表
```sql
trino:testdb_iceberg> CREATE TABLE iceberg.testdb_iceberg.tab1(id int, name varchar,age INT);
CREATE TABLE
trino:testdb_iceberg> insert into iceberg.testdb_iceberg.tab1 values(1,'lily',13),(2,'lucy',13),(3,'lintao',14);
INSERT: 3 rows
trino:testdb_iceberg>  alter table tab1 rename to iceberg.testdb_iceberg.tab1_1;
RENAME TABLE
```
只修改HMS中的元数据信息。不涉及iceberg自己的元数据和数据文件。重命名表后，iceberg磁盘文件中的表名称没有变，还是原来的名字<br />

<a name="Yalhy"></a>
## 1.2 新增列字段
修改iceberg表结构-增加列字段
```sql
trino:testdb_iceberg> alter table tab1_1 add column if not exists gender varchar;
ADD COLUMN
```
生成了新的iceberg的元数据文件*.json，如下图所示,在schemas中添加了一个字段。<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051273833-d17af5ee-8125-4584-95fc-8f7d27522138.png#id=dv4EI&originHeight=799&originWidth=1828&originalType=binary&ratio=1&status=done&style=none)<br />

<a name="iiXPl"></a>
## 1.3 重命名列字段
```sql
trino:testdb_iceberg> alter table tab1_1 rename column gender to Gen;
RENAME COLUMN
```
生成了新的了iceberg的元数据文件*.json，如下图所示。<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051274404-fa6543ed-b9ca-4538-a8e6-b3c21e2dca98.png#id=nVhqE&originHeight=580&originWidth=1802&originalType=binary&ratio=1&status=done&style=none)
<a name="yN4Hl"></a>
## 1.4 删除列字段
```sql
trino:testdb_iceberg>  alter table tab1_1 drop  column gen;
DROP COLUMN
```
生成新的iceberg的元数据文件*.json，在schemas中删除了一个字段。<br />

<a name="RKbx9"></a>
## 1.5 修改字段类型
spark-sql 修改字段类型(只支持类型提升)<br />类型提升：int->long,  float->double, decimal(p,n)->decimal(p,n+1);<br />
<br />使用spark-3.0
```sql
./spark-sql --packages org.apache.iceberg:iceberg-spark3-runtime:0.11.1     --conf spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog     --conf spark.sql.catalog.spark_catalog.type=hive
spark-sql> alter table tab1_1 alter column id type long;
```
<a name="Dd9ec"></a>
# 2 时间旅行
新插入一条数据
```sql
trino:testdb_iceberg> insert into tab1_1 values(4,'lilei',14);
INSERT: 1 row
```
![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051274898-743ca400-1ca2-471f-b053-9a40c39d46ca.png#height=105&id=DiTVL&originHeight=213&originWidth=556&originalType=binary&ratio=1&status=done&style=none&width=274)
<a name="Ri0Qp"></a>
## 2.1 查看快照id
连接器为每个Iceberg表提供一个系统快照表。快照由BIGINT快照id标识。<br />1）通过命令查询tab1_1表的最新快照ID。
```sql
trino:testdb_iceberg>  SELECT snapshot_id FROM "tab1_1$snapshots" ORDER BY committed_at DESC LIMIT 1;
     snapshot_id     
---------------------
 3198841818357182998 
(1 row)

Query 20210604_170515_00062_g8abm, FINISHED, 1 node
Splits: 18 total, 18 done (100.00%)
0.85 [3 rows, 1.36KB] [3 rows/s, 1.6KB/s]
```

<br />2）查看所有快照id信息。
```sql
trino:testdb_iceberg>  SELECT snapshot_id FROM "tab1_1$snapshots" ORDER BY committed_at DESC;
     snapshot_id     
---------------------
 3198841818357182998 
 5012893412492853402 
 6009592818754377053 
(3 rows)

Query 20210604_170526_00063_g8abm, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.74 [3 rows, 1.36KB] [4 rows/s, 1.85KB/s]
```


<a name="yYMbi"></a>
## 2.2 回溯到历史快照
使用SQL precedure Rollback_to_snapshot可以将表的状态回滚到历史指定快照id:
```sql
trino:testdb_iceberg> CALL system.rollback_to_snapshot('testdb_iceberg', 'tab1_1', 5012893412492853402);
CALL
```
![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051275422-4115e045-3d09-4dd3-886f-87cf0f9d59c0.png#height=126&id=Cj99t&originHeight=386&originWidth=1276&originalType=binary&ratio=1&status=done&style=none&width=416)
<a name="c8Vjx"></a>
# 3 表分区
<a name="oTxcG"></a>
## 3.1 分区类型
支持的分区类型如下所示：<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051275930-410d73fe-35dc-48ec-838e-c84d4b996529.png#id=KF9wR&originHeight=587&originWidth=1264&originalType=binary&ratio=1&status=done&style=none)<br />说明：<br />其中identity可以理解为list分区。<br />Bucket[N]可以理解为hash分区。<br />Year,month,day,hour为range分区。<br />

<a name="0sxQ5"></a>
## 3.2 示例
在这个例子中，表是按order_date的月份、account_number的散列(包含10个桶)和country进行分区的，<br />1）建表语句如下：
```sql
CREATE TABLE iceberg.testdb_iceberg.customer_orders (
    order_id BIGINT,
    order_date DATE,
    account_number BIGINT,
    customer VARCHAR,
    country VARCHAR)
WITH (partitioning = ARRAY['month(order_date)', 'bucket(account_number, 10)', 'country'])
```

1. 插入数据：
```sql
insert into iceberg.testdb_iceberg.customer_orders select a.orderkey as order_id , a.orderdate as order_date , cast(a.totalprice as BIGINT ) as account_number , b.name as customer, c.name as country from tpch.sf1.orders a , tpch.sf1.customer b , tpch.sf1.nation c where a.custkey=b.custkey and c.nationkey=b.nationkey;
```

<br />数据文件的目录接口如下所示：<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051276421-8017e9e8-7186-4a13-a3e7-0a84e6dec9cf.png#id=xdd77&originHeight=471&originWidth=1193&originalType=binary&ratio=1&status=done&style=none)<br />

<a name="UgjDI"></a>
# 4 分区演化
<a name="2Azti"></a>
## 4.1 创建分区表
1)按照年份对订单进行分区。
```sql
 CREATE TABLE iceberg.testdb_iceberg.orders (         
    orderkey bigint NOT NULL,           
    custkey bigint NOT NULL,            
    orderstatus varchar(1) NOT NULL,    
    totalprice double NOT NULL,         
    orderdate date NOT NULL,            
    orderpriority varchar(15) NOT NULL, 
    clerk varchar(15) NOT NULL,         
    shippriority integer NOT NULL,      
    comment varchar(79) NOT NULL        
 )WITH (partitioning = ARRAY['year(orderdate)'])
```
​

2）插入数据
```sql
 insert into iceberg.testdb_iceberg.orders select * from tpch.sf1.orders;
INSERT: 1500000 rows
Query 20210525_115053_00055_rkc2z, FINISHED, 1 node
Splits: 36 total, 36 done (100.00%)
18.46 [1.5M rows, 0B] [81.2K rows/s, 0B/s]
```

<br />3）查看分区统计信息
```sql
SELECT * FROM "orders$partitions"
```
可以看到每个分区中各个列字段的统计信息。<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051276912-f3b40267-49e3-41e5-88e5-ca5fb922ec2b.png#height=218&id=E69gf&originHeight=654&originWidth=1130&originalType=binary&ratio=1&status=done&style=none&width=377)<br />

<a name="Hz25c"></a>
## 4.2 新增month分区
1)Spark3.0操作iceberg表，新增分区和删除分区。
```sql
spark-sql> alter table orders add partition field months(orderdate);
```
![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051277406-76bf6221-b2bb-4019-820a-3cb1c2439ca9.png#height=237&id=CYID6&originHeight=472&originWidth=739&originalType=binary&ratio=1&status=done&style=none&width=371)<br />
<br />2)插入一条记录数据：
```sql
trino:testdb_iceberg> insert into orders values(3000001,145618,'F',30175.88, date '1992-12-17','4-NOT SPECIFIED','Clerk#000000141',0,'l packages. furiously careful instructions grow furi') ;
```
对应的数据文件：<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051277943-eccc3ed8-41fe-4cdc-80e7-4361d4035376.png#id=EM5bJ&originHeight=395&originWidth=1218&originalType=binary&ratio=1&status=done&style=none)<br />

1. 插入一个月的记录
```sql
trino:testdb_iceberg> insert into orders select * from tpch.sf1.orders where orderdate > date '1993-12-01' and orderdate < date '1993-12-31' ;
```
![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051278582-07439e05-9f3f-4a6b-b5eb-9305713bb64d.png#id=BeOJU&originHeight=140&originWidth=1460&originalType=binary&ratio=1&status=done&style=none)<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051279009-ae848244-f41b-4777-8f31-f24896f001be.png#id=s84Ga&originHeight=114&originWidth=1463&originalType=binary&ratio=1&status=done&style=none)<br />查询分区统计信息。
```sql
 select * from  "orders$partitions";
```
可以看到新增了orderdate_month列的统计信息。根据上述的内容插入了1993-12月份的记录和1992-12-17的记录后，orderdate_year，row_count列数据的变化<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051279591-6e0f0234-9621-44bb-83db-ad1ff9dd2106.png#id=LK5xg&originHeight=354&originWidth=1897&originalType=binary&ratio=1&status=done&style=none)<br />
<br />对应的数据文件：<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051280093-c5993372-f6ef-4ea2-a281-b41aba954de4.png#height=199&id=iRjpc&originHeight=463&originWidth=1221&originalType=binary&ratio=1&status=done&style=none&width=526)
<a name="sNce2"></a>
# 5 删除分区
<a name="xvVPb"></a>
## 5.1 DDL
通过Spark3.0 alter table语句删除range分区：
```sql
spark-sql>  ALTER TABLE orders  drop PARTITION FIELD years(orderdate);
```
只修改json文件，删除了分区机制。数据文件不受影响。<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051280669-734c7a36-6172-47aa-b38b-cb6efc632b30.png#id=k8XoK&originHeight=557&originWidth=1822&originalType=binary&ratio=1&status=done&style=none)<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051281222-7df96cfc-eed3-4c6a-9e3c-5c45910ac79d.png#id=dP2p8&originHeight=721&originWidth=1817&originalType=binary&ratio=1&status=done&style=none)<br />

<a name="opL1z"></a>
## 5.2 DML
通过trino操作：<br />1)建分区表，使用identity类型分区。即表的列字段内容
```sql
CREATE TABLE iceberg.testdb_iceberg.test_p (
  id int,
  name varchar,
  gender varchar)
WITH (partitioning = ARRAY[ 'gender']);
```
2）插入记录
```sql
trino:testdb_iceberg> insert into test_p values(1,'lily','M'),(2,'lucy','M'),(3,'lintao','F'),(4,'lilei','F');
```

<br />3）删除分区
```sql
trino:testdb_iceberg> delete from test_p where gender='F';
```
![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051281743-fa441205-893b-49ef-85ae-bae4215a0a85.png#height=185&id=JuXys&originHeight=331&originWidth=720&originalType=binary&ratio=1&status=done&style=none&width=403)<br />
<br />与spark3.0 alter table不同的是，这里标记删除了数据。查看json文件：<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051282237-b5b4a1d5-3467-4751-b030-15e80e652153.png#id=Jih0o&originHeight=580&originWidth=1876&originalType=binary&ratio=1&status=done&style=none)<br />如上图所示，分区信息没有变化，新生成了快照文件，将对应分区的文件标记删除了。总结如下：<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051282919-6d2ea85e-acc2-4193-acae-08bda3520394.png#height=339&id=SmnnH&originHeight=742&originWidth=1315&originalType=binary&ratio=1&status=done&style=none&width=601)<br />
<br />

<a name="nCrBt"></a>
# 6 row level delete
![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051283429-79ec7b2f-e565-42cf-a3eb-2c10f73a8890.png#height=129&id=F60fe&originHeight=273&originWidth=837&originalType=binary&ratio=1&status=done&style=none&width=395)<br />Spark3.0+iceberg支持row level delete，需要配置：<br />spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions<br />可以通过在spark-sql client启动参数中通过--conf 方式追加，也可以通过配置/usr/local/spark/conf/spark-defaults.conf文件中添加配置参数：<br />spark.sql.extensions org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions<br />
<br />【说明】SQL extensions are not available for Spark 2.4.<br />**启动spark-sql:（在启动参数中加入spark.sql.extensions配置）**
```sql
 ./spark-sql --packages org.apache.iceberg:iceberg-spark3-runtime:0.11.1     --conf spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog     --conf spark.sql.catalog.spark_catalog.type=hive --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
spark-sql> use testdb_iceberg;
spark-sql>  delete from tab1_1 where  id=4;
```
<a name="gjOHC"></a>
# 7 Upsert
Spark 3增加了对MERGE INTO查询的支持，可以表示行级更新。<br />Iceberg通过重写包含需要在overwrite commit中更新的行的数据文件来支持MERGE INTO。<br />MERGE INTO is recommended instead of INSERT OVERWRITE because Iceberg can replace only the affected data files, and because the data overwritten by a dynamic overwrite may change if the table’s partitioning changes.
<a name="QGB6O"></a>
## 7.1 Merge into语法
MERGE INTO updates a table, called the target table, using a set of updates from another query, called the source. The update for a row in the target table is found using the ON clause that is like a join condition.<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051283805-00b30861-3385-472b-b9d2-56bc79d32aea.png#id=mV1vv&originHeight=182&originWidth=1270&originalType=binary&ratio=1&status=done&style=none)<br />Updates to rows in the target table are listed using WHEN MATCHED ... THEN .... Multiple MATCHED clauses can be added with conditions that determine when each match should be applied. The first matching expression is used.<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051284244-9da13935-4684-4e46-b501-d24a35c979b2.png#id=HVdsl&originHeight=153&originWidth=1286&originalType=binary&ratio=1&status=done&style=none)<br />![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051284815-f2673cf9-6353-4f70-9383-298b22c0e74c.png#id=TpJ9I&originHeight=373&originWidth=1307&originalType=binary&ratio=1&status=done&style=none)<br />【注意】使用merge into时，根据on条件筛选的源表的记录只能是一条，如果是多条将会抛出异常：<br />org.apache.spark.SparkException: The ON search condition of the MERGE statement matched a single row from the target table with **multiple rows of the source table**. This could result in the target row being operated on more than once with an update or delete operation and is not allowed.
<a name="5SFnh"></a>
## 7.2 示例
Spark3.0+iceberg支持merge into ，需要配置：<br />spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions<br />**启动spark-sql:**
```sql
 ./spark-sql --packages org.apache.iceberg:iceberg-spark3-runtime:0.11.1     --conf spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog     --conf spark.sql.catalog.spark_catalog.type=hive --conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions
```


<a name="JrBCi"></a>
### 7.2.1 Update操作：
```sql
spark-sql>  use testdb_iceberg;
spark-sql> merge into tab1_1 t using (select 1 as id,'lily' as name,13 as age) s on t.id=s.id when matched and t.age=13  then update set t.age=12;
```
![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051285220-2fd182b1-8e29-4d56-9418-13783371bac9.png#height=192&id=FwFFL&originHeight=636&originWidth=898&originalType=binary&ratio=1&status=done&style=none&width=271)
<a name="TAXoC"></a>
### 7.2.2 Insert操作
```sql
spark-sql> merge into tab1_1 t using (select 5 as id,'hameimei' as name,13 as age) s on t.id=s.id when not matched then insert *;
```
![](https://cdn.nlark.com/yuque/0/2021/png/20359301/1623051285781-b8635ce3-b4ee-4083-b2a7-7164379e8f6d.png#height=286&id=Uula9&originHeight=664&originWidth=684&originalType=binary&ratio=1&status=done&style=none&width=295)<br />

<a name="gq4sX"></a>
# 8 快照管理
待补充
<a name="fMg1F"></a>
# 9 小文件合并
待补充<br />

