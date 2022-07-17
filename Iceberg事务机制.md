在应用iceberg表格式进行数据存储时，上层业务写好一批文件，调用iceberg的commit接口提交本次写入形成一个新的snapshot快照。

写引擎调用iceberg的commit接口，iceberg主要会做如下几个事情：

1. 根据提交的文件解析出对应的文件元数据生成一个manifest文件，manifest文件中包含所有提交的数据文件的统计信息，每个数据文件在manifest文件中就是一条记录。
2.  manifest文件生成之后，会紧接着生成一个manifests（snapshot快照）文件。manifests文件中每条记录是这个表当前所有manifest文件统计信息集合。每个manifest文件在manifests文件中就是一条记录。

3. manifests（snapshot快照）文件生成之后，再紧接着生成一个表元数据版本文件（文件名为：xxxx-metadata.json，其中xxxx是当前snapshot的版本号）。表元数据版本文件记录这个snapshot对应的表schema信息、partition spec信息以及manifests文件的路径等。


需要说明的是，整个commit过程是一个事务执行，即实现了ACID特性。
**原子性**：整个提交要么成功，要么失败。不会存在中间过程。
**一致性**：事务提交成功之后表的snapshot会从一个版本变更为另一个版本。
**隔离性**：一旦提交成功之后其他查询服务才可以查询到数据，否则查询不到。
**持久性**：事务提交之后，数据会被永久性地持久化到存储系统。


如何实现多线程并发场景下的ACID，以应用Hadoop Catalog的iceberg表为例进行说明：

每个iceberg表都有一个HDFS文件记录这个表的当前snapshot版本，文件称为version-hint.text。

1. commit开始之后读取version-hint.text文件中记录的当前snapshot版本，称为base-version。

2. 基于当前base-version加1生成new-version，在tmp目录下生成一个新的snapshot文件，命名为{new-version}-metadata.json。

3. 将这个tmp目录下的snapshot文件**rename**到表的metadata目录下。<br />整个commit过程利用了乐观锁，以及HDFS rename操作的原子性保证ACID事务性。


# 事务操作相关接口
本节，主要说明与表（事务）操作相关的几个主要接口和类的UML图。

针对iceberg表，可以分为表的单语句操作和表的多语句操作，其分别对应两种表的操作接口：
1）表接口，interface Table，这是对表的单语句操作的接口。
2）事务接口，interface Transaction，这是对表执行多语句操作时需要的接口。可以理解为“狭义的事务”。

## 表单语句操作接口
针对表的单语句操作，应用接口interface Table。Table接口和实现类BaseTable的关系：<br />![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721199056-bd5fe5ae-40eb-45d6-9d3b-0287b568090c.png#clientId=uc988fdec-5084-4&from=paste&height=365&id=uefbf6057&margin=%5Bobject%20Object%5D&name=image.png&originHeight=729&originWidth=390&originalType=binary&ratio=2&size=31625&status=done&style=none&taskId=u7d1a93db-ba3a-4bf4-a261-41fa820e279&width=195)

<a name="7htYW"></a>
## 表多语句操作接口
针对表的多语句操作，应用接口interface Transaction，接口有不同的实现类，包括BaseTransaction和CommitCallbackTransaction，其关系如下：![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721247596-f1a3994f-e55c-45ba-a9c8-c6272d31d55a.png#clientId=uc988fdec-5084-4&from=paste&height=440&id=u8cc9a9d1&margin=%5Bobject%20Object%5D&name=image.png&originHeight=880&originWidth=637&originalType=binary&ratio=2&size=63638&status=done&style=none&taskId=u6358d045-a20b-480e-9aba-93f54cded9b&width=318.5)


## 事务类型
根据iceberg表的存在状态，可以创建四种类型的事务，其类型如下：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721267964-b07a905f-0959-4db8-a4e7-d79bea6a6800.png#clientId=uc988fdec-5084-4&from=paste&height=122&id=u04bd606f&margin=%5Bobject%20Object%5D&name=image.png&originHeight=243&originWidth=511&originalType=binary&ratio=2&size=11384&status=done&style=none&taskId=u6a642141-24e9-4714-b5cd-68ac0509fb3&width=255.5)


## 表操作接口
在iceberg表commit操作过程中，一个重要的接口是：interface TableOperations。这是用于抽象表的元数据访问和更新的SPI接口。<br />不同的实现，针对元数据的管理方式不同，如针对Hive Catalog的类HiveTableOperations，以及针对Hadoop Catalog的类HadoopTableOperations。关系图如下所示：<br />![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721294889-c5c97d3c-4587-42ec-9f68-9f39428de6a1.png#clientId=uc988fdec-5084-4&from=paste&height=316&id=u98cc811e&margin=%5Bobject%20Object%5D&name=image.png&originHeight=631&originWidth=714&originalType=binary&ratio=2&size=45766&status=done&style=none&taskId=ub9803541-b006-42bb-af9c-d48b8ebba50&width=357)

<a name="0dEPe"></a>
## 快照文件生成接口
针对表的操作，涉及iceberg表数据的操作，iceberg会为本次操作生成对应的Snapshot快照，也就是manifest文件和snapshot（manifest list）文件。<br />针对表属性的更新操作，仅更新表的元数据信息，并生成对应版本的元数据文件，不会生成对应的snapshot快照文件。<br />这两种针对表的更新操作，应用统一的接口interface PendingUpdate<T>，相关的类关系如下：<br />![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721340136-95785dc1-4f21-477f-80e2-84ca4b7b75db.png#clientId=uc988fdec-5084-4&from=paste&height=428&id=u443fd65a&margin=%5Bobject%20Object%5D&name=image.png&originHeight=855&originWidth=751&originalType=binary&ratio=2&size=83094&status=done&style=none&taskId=u2199ab99-b7fd-45b9-9d4c-dff50967894&width=375.5)

<a name="PTDZ9"></a>
# 事务操作基本流程
下面，介绍表操作过程中的涉及到事务的操作流程图。
<a name="vaNY0"></a>
## 事务创建
这里的事务，指的是表的多语句操作。基本的流程如下图所示：<br />构建一个interface Transaction对象。<br />![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721364316-f40ec64c-9e96-4ab5-b5c4-a94ac391f66c.png#clientId=uc988fdec-5084-4&from=paste&height=380&id=ua100f262&margin=%5Bobject%20Object%5D&name=image.png&originHeight=760&originWidth=622&originalType=binary&ratio=2&size=54616&status=done&style=none&taskId=ud62819fa-b479-4299-ba80-681de5ebf69&width=311)

<a name="PFDVp"></a>
## 事务commit操作
需要说明的是，表的单语句操作，其commit过程，就是一个事务过程。<br />![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721383547-a828ec9f-3602-40fb-96e9-f5894ecd305c.png#clientId=uc988fdec-5084-4&from=paste&height=410&id=uafe25c77&margin=%5Bobject%20Object%5D&name=image.png&originHeight=820&originWidth=867&originalType=binary&ratio=2&size=72865&status=done&style=none&taskId=u743af721-d53d-44b5-9131-c89915df11d&width=433.5)


<a name="dpAGk"></a>
## 快照文件生成
在表的操作中，针对表数据每一次更新操作，都会执行PendingUpdate.commit操作产生新版本的Snapshot快照文件，即manifest文件和snapshot（manifest list）文件。而快照生成类为SnapshotProducer。<br />另外，针对表的单语句操作和表的多语句操作，在生成snapshot快照文件的过程是一致的，只是在后续对表的metadata数据的更新过程中存在差异。针对单语句操作，直接应用对应的TableOperations实现类更新表的最新metadata数据；而针对多语句操作，应用的TableOperations实现类为BaseTransaction. TransactionTableOperations，其commit操作，仅仅更新当前事务中表最新metadata数据。该提交操作的过程：<br />![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721406310-bee79d1e-3b65-4b36-9bc8-0f9b719e2780.png#clientId=uc988fdec-5084-4&from=paste&height=422&id=u44a48570&margin=%5Bobject%20Object%5D&name=image.png&originHeight=843&originWidth=582&originalType=binary&ratio=2&size=51622&status=done&style=none&taskId=ueadb1993-6df8-4fb4-b9ce-65ea7a0f169&width=291)


<a name="ZP3eq"></a>
## 事务冲突处理
在事务的提交过程中，如果由于事务操作过程中，其它操作对表进行了更新，即该事务操作的base metadata发生了变化，此时，进行事务提交时，就会发生冲突。发生冲突时的操作过程：<br />![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721436074-6734dacf-bb07-430f-b99c-b56794f523b0.png#clientId=uc988fdec-5084-4&from=paste&height=222&id=u7165427c&margin=%5Bobject%20Object%5D&name=image.png&originHeight=444&originWidth=346&originalType=binary&ratio=2&size=20426&status=done&style=none&taskId=ub207754c-4c39-4ac4-b645-b2cd023ca24&width=173)

例如，事务中对表进行了更新操作，产生新版本的snapshot，以及对应的manifest文件。如下图中的snap-1-1-xxx。在事务操作过程中，另一个用户对表进行更新操作，产生了对应的snapshot，如下图的snap-2-1-xxx。<br />![](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721034945-7517007b-b2af-4837-b02c-5370b3bb4ef3.png#id=eXaqw&originHeight=139&originWidth=865&originalType=binary&ratio=1&status=done&style=none)<br />此时，提交事务，就会产生冲突。<br />发生冲突的事务，会读取表最新的metadata数据，获取对应的Snapshot信息，然后基于新的表元数据信息，重新对该事务中执行的updates操作进行提交，即：PendingUpdate.comitt()，进行snapshot的Merge操作，产生新版本的snapshot信息和对应的snapshot文件，如下图所示：<br />![](https://cdn.nlark.com/yuque/0/2021/png/20360202/1624721035475-da7f8a50-1a56-45ef-b797-c1f574aa39e7.png#id=e080w&originHeight=154&originWidth=837&originalType=binary&ratio=1&status=done&style=none)<br />如上图，元数据文件snap-1-2-xxx是Merge后的snapshot文件，因为数据文件没有变化，所以，不会产生新版本的manifest文件。


## 事务提交retry
在TableOperations.commit过程中，如果发生失败的情况，会根据retry的设置，进行事务的提交重试操作。基本的retry参考“事务commit操作”流程。



## 基于乐观锁的事务commit过程
iceberg中，基于乐观锁机制创建表操作（事务）过程中的snapshot文件，并进行commit操作。同时，会应用Hive Metastore或者Hadoop HDFS自身的原子性，保证元数据文件的原子操作。<br />下图，以HMS管理iceberg元数据入口为例，描述表操作（事务）的commit过程：<br />![image.png](https://cdn.nlark.com/yuque/0/2021/png/20360202/1625647743475-677a48bd-ba1b-497b-90d4-2557f27224e8.png#clientId=ufd28fa3b-634d-4&from=paste&height=361&id=u19d2e46d&margin=%5Bobject%20Object%5D&name=image.png&originHeight=721&originWidth=847&originalType=binary&ratio=1&size=77288&status=done&style=none&taskId=u8e4f824f-8704-4a1b-91eb-0e1ec1480a0&width=423.5)


