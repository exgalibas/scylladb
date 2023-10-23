## 替换ScyllaDB集群中的多个死节点
Scylla是一个容错系统，即使多个节点关闭，群集也可用

### 前提
- 执行`nodetool status`检查集群中各节点的状态，如果是`DN`则表示节点故障需要被替换/重新启动起来

    ```
    Datacenter: DC1
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    DN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
    DN  192.168.1.203  124.42 KB  256     32.6%             675ed9f4-6564-6dbd-can8-43fddce952gy   B1
    ```
    登录到某个状态为`UN`的节点，收集一下信息：
    - cluster_name - cat /etc/scylla/scylla.yaml | grep cluster_name
    - seeds - cat /etc/scylla/scylla.yaml | grep seeds:
    - endpoint_snitch - cat /etc/scylla/scylla.yaml | grep endpoint_snitch
    - Scylla version - scylla --version
    - consistent_cluster_management - grep consistent_cluster_management /etc/scylla/scylla.yaml

### 步骤
依赖副本因子
- 如果故障节点数小于keyspace的副本因子，则仍然至少有一个包含数据的可用副本，参考[替换ScyllaDB集群中的单个死节点](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/replace-dead-node.html)
- 如果故障节点数大于或等于keyspace的副本因子，则部分数据可能已经丢失，需要使用备份恢复。先操作[替换ScyllaDB集群中的单个死节点](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/replace-dead-node.html)再操作[从备份中恢复数据](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/backup-restore/restore.html)