## Scylla 和 Apache Cassandra兼容性

最新更新版本：ScyllaDB 5.0

ScyllaDB是Apache Cassandra 3.11的直接替代品，具有Apache Cassandra 4.0的附加功能。本章包含有关ScyllaDB与Apache Cassandra兼容性相关信息

下面的表格包含有关ScyllaDB开源支持Apache Cassandra功能的信息。其中不包括在Apache Cassandra中没有匹配的ScyllaDB企业专用功能或ScyllaDB特定功能。有关ScyllaDB功能的更多信息，请参阅[ScyllaDB功能](https://opensource.docs.scylladb.com/stable/using-scylla/features.html)

### 如何阅读表格

- y - 表示ScyllaDB中可用，并与Apache Cassandra兼容

- n - 表示在ScyllaDB中不可用

- nc - 表示在ScyllaDB中可用，但与Apache Cassandra不兼容


### 接口

| Cassandra接口 | ScyllaDB 支持的版本 | 备注  |
| --- | --- | --- |
| CQL | 与版本3.3.1完全兼容，具有更高CQL版本中的其他功能（例如，[Duration类型](https://opensource.docs.scylladb.com/stable/cql/types.html#durations)），与协议v4完全兼容，具有v5的附加功能 |     |
| Thrift | 与Cassandra 2.1兼容 |     |
| SSTable format (all versions) | 3.11(mc / md / me), 2.2(la), 2.1.8 (ka) | `me` - 在ScyllaDB开源5.1和ScyllaDB企业版2022.2.0（及更高版本）中被支持 <br/>`md` - 在ScyllaDB开源4.3 和ScyllaDB企业版2021.1.0（及更高版本）中被支持 |
| JMX | 3.11 |     |
| Configuration (cassandra.yaml) | 3.11 |     |
| Log | nc  |     |
| Gossip and internal streaming | nc  |     |
| SSL | nc  |     |

### 支持的工具

这些工具基于Apache Cassandra 3.11

- [Nodetool Reference](#Nodetool Reference) - 用于管理Scylla节点或集群的命令行

- [CQLSh - the CQL shell](https://opensource.docs.scylladb.com/stable/cql/cqlsh.html)

- [REST - Scylla REST/HTTP Admin API](https://opensource.docs.scylladb.com/stable/operating-scylla/rest.html)

- [Tracing](https://opensource.docs.scylladb.com/stable/using-scylla/tracing.html) - 用于调试和分析服务器内部流的ScyllaDB工具

- [SSTableloader](https://opensource.docs.scylladb.com/stable/operating-scylla/admin-tools/sstableloader.html) - 将sstables批量加载到Scylla集群

- [Scylla SStable](https://opensource.docs.scylladb.com/stable/operating-scylla/admin-tools/scylla-sstable.html) - 验证和转储SStable的内容，生成直方图，转储SStable索引的内容

- [Scylla Types](https://opensource.docs.scylladb.com/stable/operating-scylla/admin-tools/scylla-types.html) - 检查SStables、日志、coredumps等原始值

- [cassandra-stress](https://opensource.docs.scylladb.com/stable/operating-scylla/admin-tools/cassandra-stress.html) - 对ScyllaDB和cassandra进行基准测试和负载测试

- [SSTabledump](https://opensource.docs.scylladb.com/stable/operating-scylla/admin-tools/sstabledump.html) - Scylla 3.0、Scylla企业版2019.1或者更高版本支持该工具

- [SSTable2JSON](https://opensource.docs.scylladb.com/stable/operating-scylla/admin-tools/sstable2json.html) - Scylla 2.3或更低版本支持该工具

- sstablelevelreset - 使用分级压缩策略（LCS），将一组选定的SSTable级别重置为0

- sstablemetadata - 打印指定SSTable的相关元数据

- sstablerepairedset - 将特定的SSTables标记为已修复或未修复

- configuration_encryptor - 使用系统密钥[静态加密](https://opensource.docs.scylladb.com/stable/operating-scylla/security/encryption-at-rest.html)敏感scylla配置项

- local_file_key_generator - 生成用于[静态加密](https://opensource.docs.scylladb.com/stable/operating-scylla/security/encryption-at-rest.html)的本地文件（系统）密钥，支持用户提供长度、密钥算法、算法块模式和算法填充方法

- [scyllatop](https://www.scylladb.com/2016/03/22/scyllatop/) - 用于scylladb收集prometheus指标的基于终端类似top的工具

- [scylla_dev_mode_setup](https://opensource.docs.scylladb.com/stable/getting-started/install-scylla/dev-mod.html) - 在开发模式下运行Scylla

- [perftune](https://opensource.docs.scylladb.com/stable/operating-scylla/admin-tools/perftune.html) - 性能配置

使用-h，--help运行每个工具以获取完整的选项说明

### 特性

#### 一致性级别（读取和写入）

| Options | Support |
| --- | --- |
| Any (Write Only) | y   |
| One | y   |
| Two | y   |
| Three | y   |
| Quorum | y   |
| All | y   |
| Local One | y   |
| Local Quorum | y   |
| Each Quorum (Write Only) | y   |
| SERIAL | y*  |
| LOCAL_SERIAL | y*  |

后面带*的表示从ScyllaDB 4.0开始支持

#### Snitches

| Options | Support |
| --- | --- |
| [SimpleSnitch](https://opensource.docs.scylladb.com/operating-scylla/system-configuration/snitch/#simplesnitch) | y   |
| [RackInferringSnitch](https://opensource.docs.scylladb.com/operating-scylla/system-configuration/snitch/#rackinferringsnitch) | y   |
| PropertyFileSnitch | n   |
| [GossipingPropertyFileSnitch](https://opensource.docs.scylladb.com/operating-scylla/system-configuration/snitch/#gossipingpropertyfilesnitch/) | y   |
| Dynamic snitching | n   |
| [EC2Snitch](https://opensource.docs.scylladb.com/operating-scylla/system-configuration/snitch/#ec2snitch/) | y   |
| [EC2MultiRegionSnitch](https://opensource.docs.scylladb.com/operating-scylla/system-configuration/snitch/#ec2multiregionsnitch) | y   |
| [GoogleCloudSnitch](https://opensource.docs.scylladb.com/operating-scylla/system-configuration/snitch/#googlecloudsnitch) | y   |
| CloudstackSnitch | n   |

#### 分流器

| Options | Support |
| --- | --- |
| Murmur3Partitioner (default) | y   |
| RandomPartitioner | n*  |
| OrderPreservingPartitioner | n   |
| ByteOrderedPartitioner | n*  |
| CollatingOrderPreservingPartitioner | n   |

后面带*的表示从ScyllaDB 4.0开始移除（即不支持）

#### 协议选项

| Options | Support |
| --- | --- |
| [Encryption](https://opensource.docs.scylladb.com/operating-scylla/security/client_node_encryption/) | y   |
| [Authentication](https://opensource.docs.scylladb.com/operating-scylla/security/authentication/) | y   |
| [Compression](https://opensource.docs.scylladb.com/operating-scylla/admin/#compression) (see below) | y   |

#### 压缩

| Options | Support |
| --- | --- |
| CQL Compression | y   |
| LZ4 | y   |
| Snappy | y   |
| [Node to Node Compression](https://opensource.docs.scylladb.com/operating-scylla/admin/#internode-compression) | y   |
| [Client to Node Compression](https://opensource.docs.scylladb.com/operating-scylla/admin/#client-node-compression) | y   |

#### 备份和恢复

| Options | Support |
| --- | --- |
| [Snapshot](https://opensource.docs.scylladb.com/operating-scylla/procedures/backup-restore/backup/#full-backup-snapshots) | y   |
| [Incremental backup](https://opensource.docs.scylladb.com/operating-scylla/procedures/backup-restore/backup/#incremental-backup) | y   |
| [Restore](https://opensource.docs.scylladb.com/operating-scylla/procedures/backup-restore/restore/) | y   |

#### 修复和一致性

| Options | Support |
| --- | --- |
| [Nodetool Repair](https://opensource.docs.scylladb.com/operating-scylla/nodetool-commands/repair/) | y   |
| Incremental Repair | n   |
| [Hinted Handoff](https://opensource.docs.scylladb.com/architecture/anti-entropy/hinted-handoff/) | y   |
| [Lightweight transactions](https://opensource.docs.scylladb.com/using-scylla/lwt/) | y*  |

后面带*的表示从ScyllaDB 4.0开始支持

#### 副本替换策略

| Options | Support |
| --- | --- |
| SimpleStrategy | y   |
| NetworkTopologyStrategy | y   |

#### 安全

| Options | Support |
| --- | --- |
| Role Based Access Control (RBAC) | y   |

#### 索引和缓存

| Options | Support |
| --- | --- |
| row / key cache | n   |
| [Secondary Index](https://opensource.docs.scylladb.com/using-scylla/secondary-indexes/) | y*  |
| [Materialized Views](https://opensource.docs.scylladb.com/using-scylla/materialized-views/) | y*  |

后面带*的表示从ScyllaDB开源版本和ScyllaDB企业版2019.1之后支持

#### 附加特性

| Feature | Support |
| --- | --- |
| Counters | y   |
| User Defined Types | y   |
| User Defined Functions | n*  |
| Time to live (TTL) | y   |
| Super Column | n   |
| vNode Enable | y-Default |
| vNode Disable | n   |
| Triggers | n   |
| Batch Requests | y-包括条件更新 |

后面带*表示还处于实验中

### CQL命令兼容性

#### 创建keyspace

| Feature | Support |
| --- | --- |
| DURABLE_WRITES | y   |
| IF NOT EXISTS | y   |
| WITH REPLICATION | y(see below) |

#### 创建带副本的keyspace

| Feature | Support |
| --- | --- |
| SimpleStrategy | y   |
| NetworkTopologyStrategy | y   |
| OldNetworkTopologyStrategy | n   |

#### 创建表

| Feature | Support |
| --- | --- |
| Primary key column | y   |
| Compound primary key | y   |
| Composite partition key | y   |
| Clustering order | y   |
| Static column | y   |

#### 创建表Att

| Feature | Support |
| --- | --- |
| bloom_filter_fp_chance | y   |
| caching | n(ignored) |
| comment | y   |
| compaction | y   |
| compression | y   |
| dclocal_read_repair_chance | y   |
| default_time_to_live | y   |
| gc_grace_seconds | y   |
| index_interval | n   |
| max_index_interval | y   |
| memtable_flush_period_in_ms | n(ignored) |
| min_index_interval | y   |
| populate_io_cache_on_flush | n   |
| read_repair_chance | y   |
| replicate_on_write | n   |
| speculative_retry | `ALWAYS`, `NONE` |

#### 创建表Compaction

| Feature | Support |
| --- | --- |
| [SizeTieredCompactionStrategy](https://opensource.docs.scylladb.com/getting-started/compaction/#size-tiered-compaction-strategy) (STCS) | y   |
| [LeveledCompactionStrategy](https://opensource.docs.scylladb.com/getting-started/compaction/#leveled-compaction-strategy) (LCS) | y   |
| DateTieredCompactionStrategy (DTCS) | y*  |
| [TimeWindowCompactionStrategy](https://opensource.docs.scylladb.com/getting-started/compaction/#time-window-compactionstrategy) (TWCS) | y   |

后面带*代表从ScyllaDB 4.0开始被移除，使用TWCS代替

#### 创建表Compression

| Feature | Support |
| --- | --- |
| sstable_compression LZ4Compressor | y   |
| sstable_compression SnappyCompressor | y   |
| sstable_compression DeflateCompressor | y   |
| chunk_length_kb | y   |
| crc_check_chance | n   |

#### Alter命令

| Feature | Support |
| --- | --- |
| ALTER KEYSPACE | y   |
| ALTER TABLE | y   |
| ALTER TYPE | y   |
| ALTER USER | y   |
| ALTER ROLE | y   |

#### 数据操作

| Feature | Support |
| --- | --- |
| BATCH | y   |
| INSERT | y   |
| Prepared Statements | y   |
| SELECT | y   |
| TRUNCATE | y   |
| UPDATE | y   |
| USE | y   |

#### Create命令

| Feature | Support |
| --- | --- |
| CREATE TRIGGER | n   |
| CREATE USER | y   |
| CREATE ROLE | y   |

#### Drop命令

| Feature | Support |
| --- | --- |
| DROP KEYSPACE | y   |
| DROP TABLE | y   |
| DROP TRIGGER | n   |
| DROP TYPE | y   |
| DROP USER | y   |
| DROP ROLE | y   |

#### 角色和权限

| Feature | Support |
| --- | --- |
| GRANT PERMISSIONS | y   |
| GRANT ROLE | y   |
| LIST PERMISSIONS | y   |
| LIST USERS | y   |
| LIST ROLES | y   |
| REVOKE PERMISSIONS | y   |
| REVOKE ROLE | y   |

#### 物化视图

| Feature | Support |
| --- | --- |
| MATERIALIZED VIEW | y   |
| ALTER MATERIALIZED VIEW | y   |
| CREATE MATERIALIZED VIEW | y   |
| DROP MATERIALIZED VIEW | y   |

#### 索引命令

| Feature | Support |
| --- | --- |
| INDEX | y   |
| CREATE INDEX | y   |
| DROP INDEX | y   |