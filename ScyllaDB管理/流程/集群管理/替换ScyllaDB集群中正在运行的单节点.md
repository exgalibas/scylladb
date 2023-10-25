## 替换ScyllaDB集群中正在运行的单节点

有如下两种方式可以替换ScyllaDB集群中正在运行的节点

### 添加新节点并停用旧节点

添加新节点，然后停用旧节点，群集在整个过程中保持相同级别的数据复制，代价是在此过程中会传输更多数据。将新节点添加到Scylla群集时，为了确保数据再次均匀分布，其他节点会将相关数据流式传输到新节点。从Scylla集群中停用节点时，为了确保数据再次均匀分布，此节点会将其数据流式传输到其他节点。因此，通过添加和停用来替换节点总共会发生两次数据流传输

#### 步骤

1. 参考[添加一个新节点到现有Scylla集群](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/add-node-to-cluster.html)
2. 参考[从Scylla集群中移除节点](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/remove-node.html)，停用旧节点
3. 在所有剩余的节点中执行`nodetool cleanup`
4. 通过`nodetool status`确认目标节点已停用

### 直接替换

直接停掉节点并替换，流程中集群的数据副本因子会短暂减少，这可能是不安全的，但是这种方式更有效率，且整个流程只发生一次数据流传输

#### 步骤

1. 执行`nodetool drain`（该命令会切断当前节点与Scylla集群的连接）
2. 停止Scylla服务

    支持的OS
    ```
    sudo systemctl stop scylla-server
    ```
    Docker
    ```
    docker exec -it some-scylla supervisorctl stop scylla
    ```
3. 参考[替换ScyllaDB集群中的单个故障节点](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/replace-dead-node.html)
4. 执行`nodetool status`确认是否替换成功