## 从Apache Cassandra迁移到Scylla

注意：以下说明适用于从Apache Cassandra迁移到Scylla，而不是从DataStax Enterprise迁移。DataStax Enterprise SSTable格式与Apache Cassandra或Scylla SSTable载入程序不兼容，可能无法正确迁移

将数据从Apache Cassandra迁移到最终一致的数据存储（如Scylla）来支持高容量、低延迟的应用并验证其一致性是一个多步骤的过程，它涉及以下高级步骤：

1. 在Scylla中创建与Apache Cassandra相同的schema，尽管可能会有一些变化

2. 配置应用程序以执行双写入（仍然仅从Apache Cassandra读取）

3. 从Apache Cassandra获得所有要迁移数据的快照

4. 使用Scylla sstableloader工具将SSTable文件加载到Scylla + 数据验证

5. 验证周期：双重写入和读取，Scylla服务读取，记录不匹配直到达到最小数据不匹配阈值

6. Apache Cassandra生命周期结束：仅使用Scylla读写


注意：步骤2和5仅适用于实时迁移（意味着持续流量且没有停机时间）

![](../../图片/2023-08-08-11-26-04-image.png?msec=1697423718296)

**双重写入**：更新应用程序逻辑以写入两个数据库

![](../../图片/2023-08-08-11-27-12-image.png?msec=1697423718296)

**迁移**：将历史数据从Apache Cassandra SSTables迁移到Scylla

![](../../图片/2023-08-08-11-27-21-image.png?msec=1697423718296)

**双重读取**：持续验证两个数据库之间的数据同步

![](../../图片/2023-08-08-11-27-35-image.png?msec=1697423718391)

**实时迁移**：从DB-OLD迁移到DB-NEW时间线

### 步骤

1\. 在Scylla集群手动创建/迁移schema（键空间、表和用户定义类型），从Apache Cassandra 3.x 迁移时，需要更新部分schema[请参阅限制和已知问题部分](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cassandra-to-scylla-migration-process.html#notes-limitations-and-known-issues)

  - 从Apache Cassandra导出schema：`cqlsh [IP] “-e DESC SCHEMA” > orig_schema.cql`

  - 将schema导入Scylla：`cqlsh [IP] --file 'adjusted_schema.cql'`

  注意：Scylla和Apache Cassandra的[加密备份文件](https://opensource.docs.scylladb.com/stable/operating-scylla/security/encryption-at-rest.html)不兼容，sstableloader不支持加载加密文件，如果您需要迁移/还原加密文件：

  - 将它们上传到原始数据库

  - 使用ALTER TABLE语句解密table

  - 使用[upgradesstable](https://opensource.docs.scylladb.com/stable/operating-scylla/nodetool-commands/upgradesstables.html)更新SSTables

  - 使用sstableloader加载

  注意：建议按照如下的方式更改计划迁移的表的schema：

  - 将压缩参数min_threshold设置为2，使得通过压缩更快地消除重复。使用 sstableloader进行迁移可能会创建大量磁盘的临时副本

  - 将gc_grace_seconds（默认为十天）增加到更高的值，以确保在迁移过程中不会丢失墓碑标记（逻辑删除），建议值为315360000（10年）


  注意：Scylla开源3.0及更高版本和Scylla企业版2019.1及更高版本支持[物化视图（MV）](https://opensource.docs.scylladb.com/stable/using-scylla/materialized-views.html) 和[二级索引（SI）](https://opensource.docs.scylladb.com/stable/using-scylla/secondary-indexes.html)，使用MV或SI从Apache Cassandra迁移数据时，您可以：

  - 创建MV和SI作为schema的一部分，以便为每个新插入编制索引

  - 首先使用sstableloader上传所有数据，然后才创建[二级索引](https://opensource.docs.scylladb.com/stable/cql/secondary-indexes.html#create-index-statement)和[MVs](https://opensource.docs.scylladb.com/stable/cql/mv.html#create-materialized-view-statement)
  在任何情况下，都只使用sstableloader来加载最基本的表SSTable，不要加载索引和视图数据，应该通过Scylla为您重新生成


2\. 如果您希望在不停机的情况下执行迁移过程，请将您的应用程序配置为对Apache Cassandra和Scylla执行双重写入（请参阅下面的代码片段了解双重写入）。在执行此操作之前，请确保使用客户端生成的时间戳（写入时间），否则Scylla和Apache Cassandra上的数据会被认为是不同的，虽然数据是相同的

注意：您的应用程序应保持从Apache Cassandra读取和写入，直到整个迁移完成，数据完整性得到验证，并在执行双重写入和读取验证阶段的结果达到了您的预期

**双重写入和客户端生成的时间戳Python代码片段：**

```python
# put both writes (cluster 1 and cluster 2) into a list
    writes = []
    #insert 1st statement into db1 session, table 1
    writes.append(db1.execute_async(insert_statement_prepared[0], values))
    #insert 2nd statement into db2 session, table 2
    writes.append(db2.execute_async(insert_statement_prepared[1], values))

    # loop over futures and output success/fail
    results = []
    for i in range(0,len(writes)):
        try:
            row = writes[i].result()
            results.append(1)
        except Exception:
            results.append(0)
            #log exception if you like
            #logging.exception('Failed write: %s', ('Cluster 1' if (i==0) else 'Cluster 2'))

    results.append(values)
    log(results)

    #did we have failures?
    if (results[0]==0):
        #do something, like re-write to cluster 1
        log('Write to cluster 1 failed')
    if (results[1]==0):
        #do something, like re-write to cluster 2
        log('Write to cluster 2 failed')

    for x in range(0,RANDOM_WRITES):
        #explicitly set a writetime in microseconds
        values = [ random.randrange(0,1000) , str(uuid.uuid4()) , int(time.time()*1000000) ]
        execute( values )
```

[这里](https://github.com/scylladb/scylla-code-samples/tree/master/dual_writes)查看完整代码样例

3\. 在每个Apache Cassandra节点上，使用nodetool snapshot命令为每个keyspace生成快照，这会将所有SSTable刷新到磁盘，并为该keyspace中的每个table生成一个包含纪元时间戳的快照文件夹

发布快照的文件夹路径：`/var/lib/cassandra/data/keyspace/table-[uuid]/snapshots/[epoch_timestamp]/`

4\. 我们强烈建议不要直接在Scylla集群上运行sstableloader工具，因为它会消耗Scylla的资源。相反，您应该从中间节点运行sstableloader，为此，您需要安装scylla-tools-core软件包（它包括sstableloader工具）

  您需要确保与Apache Cassandra和Scylla集群都有连接，有两种方法可以做到这一点：

  - 选项1（推荐）：将SSTable文件从Apache Cassandra集群复制到中间节点的本地文件夹

  - 选项2：中间节点上的NFS作为Apache Cassandra节点中的SSTable文件挂载点

    - [NFS mount on CentOS](http://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-centos-6)

    - [NFS mount on Ubuntu](http://www.digitalocean.com/community/tutorials/how-to-set-up-an-nfs-mount-on-ubuntu-16-04)

    a. 安装相关pkgs后（详见上述链接），在每个Apache Cassandra节点上编辑`/etc/exports`文件，并在单独行中添加以下内容：

    `[Full path to snapshot ‘epoch’ folder] [Scylla_IP](rw,sync,no_root_squash,no_subtree_check)`

    b. 重启NFS服务 `sudo systemctl restart nfs-kernel-server`

    c. 在其中一个Scylla节点上创建一个新文件夹，并将其用作Apache Cassandra节点的挂载点

        例子：
        `sudo mount [Cassandra_IP]:[Full path to snapshots ‘epoch’ folder] /[keyspace]/[table]`
        注意：本地文件夹或NFS挂载点路径都必须以`/[keyspace]/[table]`格式结尾，为了适配sstableloader解析（有关更多详细信息，请参阅sstableloader help）

5\. 如果您不能使用中间节点（请参阅上一步），则有两个选项：

  - 选项1：将sstable文件复制到其中一个Scylla群集节点上的本地文件夹，最好在磁盘或磁盘阵列上，而不是Scylla集群RAID的一部分，但仍可供sstableloader工具访问

    注意：将其复制到Scylla RAID将需要足够的磁盘空间（Apache Cassandra SSTable快照大小x2 < Scylla节点容量的50%）来容纳复制的SSTables文件和迁移到Scylla的整个数据（还应考虑keyspace RF）

  - 选项2：Scylla节点上的NFS作为Apache Cassandra节点中的SSTable文件挂载点（请参阅上一步中的NFS挂载说明），这节省了第一个选项所需的额外磁盘空间

6\. 使用Scylla sstableloader工具（不是同名的Apache Cassandra工具）来加载SSTables，在没有任何参数的情况下运行该工具将显示选项和用法的列表，其中最重要的是SSTables目录和Scylla节点IP

  例子：

  - `sstableloader -d [Scylla IP] .../[ks]/[table]`

  - `sstableloader -d [scylla IP] .../[mount point]` (in `/[ks]/[table]` format)

7\. 建议并行运行多个sstableloader，并利用所有Scylla节点加载SSTable。从一个keyspace及其所有Apache Cassandra节点的底层SSTable文件开始，完成后继续下一个keyspace，依此类推

  注意：通过使用Throttling -t参数限制sstableloader速度，考虑您的物理硬件、实时流量负载和网络利用率（有关更多详细信息，请参阅sstableloader help）

8\. 加载完所有SSTable文件后，可以使用cqlsh或任何其他工具来验证数据是否成功迁移。强烈建议将应用程序配置为对两个数据存储执行写入和读取，跟踪这两个数据存储中的数据不匹配的请求数

9\. Apache Cassandra生命周期结束：一旦数据验证达到预期，就可以修改你的应用程序，停止对Cassandra集群的写入和读取，Scylla即成为唯一的目标/源


### 故障处理

**如果sstableloader失败了，我该怎么办？**

每次的加载失败，都需要重新加载，当您再次加载产生的部分相同的数据（在失败之前加载的数据），压缩会将这些重复处理掉

**如果Apache Cassandra节点出现故障，我该怎么办？**

如果失败的节点是正在被加载SSTable的节点，则sstableloader也将失败。如果您使用的是 RF>1，则数据也会存在于其他节点上，因此您可以继续从所有其他Cassandra节点进行加载。一旦加载完成，所有数据都应在Scylla上

**如果Scylla节点出现故障，我该怎么办？**

如果失败的节点是你正在加载sstables的节点，那么sstableloader也将失败，更换到别的Scylla节点并重新启动加载

**如何回滚并从头开始？**

1. 停止对Scylla的双重写入

2. 停止Scylla服务 `sudo systemctl stop scylla-server`

3. 使用cqlsh对已加载到Scylla的所有数据执行`truncate`

4. 再次启动对Scylla的双重写入

5. 创建所有Cassandra节点的新快照

6. 开始从新的快照文件夹再次将SSTables加载到Scylla


### 说明、限制和已知问题

1. 从Apache Cassandra 3.X迁移时，`Duration`数据类型仅在Scylla2.1版本及更高版本支持（[issue-2240](https://github.com/scylladb/scylla/issues/2240)）

2. 从Apache Cassandra 3.0迁移时，针对Scylla 2.x, 1.x对应的表schema需要进行不同的调整：

  - 创建表的变更（[issue-8384](https://issues.apache.org/jira/browse/CASSANDRA-8384)）

  - `crc_check_chance`压缩选项（[issue-9839](https://issues.apache.org/jira/browse/CASSANDRA-9839)）

3. Scylla 2.x CQL客户端cqlsh不能显示时间戳数据类型的毫秒值（[scylla-tools-java/issues #36](https://github.com/scylladb/scylla-tools-java/issues/36)）

4. 从Apache Cassandra迁移后，Scylla中的Nodetool tablestats分区键（估计）数量比Cassandra中的原始数量上下波动20%（[issue-2545](https://github.com/scylladb/scylla/issues/2545)）

5. Scylla 2.x使用的是Apache Cassandra 2.x文件格式，这意味着从Apache Cassandra 3.x迁移到Scylla 2.x将导致相同数据在Scylla集群上存储空间占用不同。Scylla 3.x使用与Cassandra 3.x相同的格式

### 计数器

在2.1版本中，Apache Cassandra改变了计数器的工作方式。以前的设计有一些难以解决的问题，这意味着没有安全和通用的方法将计数器数据从旧格式转换为新格式。因此，在版本 2.1之前创建的计数器类型数据即使在迁移到最新的Cassandra版本后也可能包含旧格式信息。由于Scylla仅实现了新的计数器设计，因此对如何从Cassandra迁移计数器类型数据加了限制

将计数器类型数据SSTables复制到Scylla是不安全的，默认情况下是不允许的。即使您使用sstableloader，它也会拒绝以旧格式加载数据

**Apache Cassandra 4.x和Scylla 4.x之间的schema差异**

下表说明了Apache Cassandra 4.x和Scylla 3.x之间的默认schema差异

显著差异：

- 由于CDC在Cassandra中的实现方式不同，因此Cassandra模式中的`cdc=false`应更改为`cdc = {'enabled':'false'}`

- `additional_write_policy = '99p'`在Scylla中不受支持，确保将其从schema中删除

- 扩展名 = {}在Scylla中不受支持，确保将其从schema中删除

- `read_repair = 'BLOCKING'`在Scylla中不支持，请确保将其从schema中删除

- 在表达式`compression = {'chunk_length_in_kb':'16', 'class':'org.apache.cassandra.io.compress.LZ4Compressor'}'`中，将`'class':'org.apache.cassandra.io.compress.LZ4Compressor'`替换为`'sstable_compression':'org.apache.cassandra.io.compress.LZ4Compressor'`

- 将`speculative_retry = '99p'`替换为`speculative_retry = '99.0PERCENTILE'`


注意：将2.1版本之前的Apache Cassandra的计数器类型数据SSTables迁移到Scylla将不起作用

**Apache Cassandra 3.x和Scylla 2.x和1.x之间的schema差异**

下表说明了Apache Cassandra 3.x和Scylla 2.x、1.x之间的默认schema差异

显著差异：

- Scylla支持'caching'部分，但需要对schema进行调整（见下文）

- `'crc_check_chance'`在Scylla中不受支持，请确保将其从schema中删除


下面是个例子，用来展示Apache Cassandra 3.x和Scylla 2.x、1.x之间具体有哪些schema的调整/删除：

| Apache Cassandra 3.10 (uses 3.x Schema) | Scylla 2.x 1.x (uses Apache Cassandra 2.1 Schema) |
| --- | --- |
| ![](../../图片/2023-08-10-17-53-57-image.png?msec=1697423718344) | ![](../../图片/2023-08-10-17-54-12-image.png?msec=1697423718345) |
| ![](../../图片/2023-08-10-17-54-38-image.png?msec=1697423718296) | 调整适配Apache Cassandra 2.1schema![](../../图片/2023-08-10-17-56-24-image.png?msec=1697423718297) |
| **AND crc_check_chance = 1.0** | 这行配置迁移到Scylla需要删除 |
| ![](../../图片/2023-08-10-17-57-58-image.png?msec=1697423718297) | ![](../../图片/2023-08-10-17-58-08-image.png?msec=1697423718297) |

更新信息请参考[Scylla和Apache Cassandra兼容性](https://opensource.docs.scylladb.com/stable/using-scylla/cassandra-compatibility.html)，也可以在Scylla大学中学习[相关课程](https://university.scylladb.com/courses/scylla-operations/lessons/migrating-to-scylla/)