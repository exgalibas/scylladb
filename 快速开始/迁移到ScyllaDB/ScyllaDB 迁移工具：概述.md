## ScyllaDB 迁移工具：概述

以下迁移工具可用于从兼容数据库迁移到Scylla，如Apache Cassandra或其他Scylla集群（开源或企业）：

- 从SSTable到SSTable

  - 基于Scylla刷新

  - 对于大规模场景，需要使用工具从一个位置上传/传输文件到另一个位置

  - 例如[unirestore](https://github.com/scylladb/field-engineering/tree/master/unirestore)

- 从SSTable到CQL

  - [sstableloader](https://opensource.docs.scylladb.com/stable/operating-scylla/admin-tools/sstableloader.html)
- 从CQL到CQL

  - [Spark Migrator](https://github.com/scylladb/scylla-migrator) - Spark迁移器允许您在将数据推送到目标数据库之前轻松转换数据
- 从DynamoDB到Scylla Alternator

  - [Spark Migrator](https://github.com/scylladb/scylla-migrator) - Spark迁移器允许您在将数据推送到目标数据库之前轻松转换数据