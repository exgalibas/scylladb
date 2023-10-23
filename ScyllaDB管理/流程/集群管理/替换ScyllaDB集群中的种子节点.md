## 替换ScyllaDB集群中的种子节点
注意：ScyllaDB开源版本4.3和企业版2021.1及后续版本中，种子节点的玩法已被去中心的gossip替代，种子节点的作用仅在启动期间新节点用于了解集群拓扑。因此无需替换在scylla.yaml文件中配置了种子节点的节点

ScyllaDB的种子节点是不会经历引导流程的(详情见https://www.scylladb.com/2020/09/22/seedless-nosql-getting-rid-of-seed-nodes-in-scylla/)，引导流程是指新节点加入后与现有节点的数据流传输。以下阐述如何替换一个故障的种子节点

### 前提
通过执行`cat /etc/scylla/scylla.yaml | grep seeds:`输出种子节点列表，确认目标节点是否在列表中

### 步骤
1. 在集群中的每个节点执行下述a-c步骤

    a.通过将节点IP添加到`/etc/scylla/scylla.yaml`文件中的种子列表中，将群集中的现有节点提升为种子节点

    b.将故障的种子节点ip移出种子节点列表，具体见[从种子列表中移除种子节点](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/remove-seed.html)

    c.重启集群

    支持的操作系统
    ```
    sudo systemctl restart scylla-server
    ```
    Docker
    ```
    docker exec -it some-scylla supervisorctl restart scylla
    ```

    注意：上述操作需要在群集中的所有节点上执行，因此需要协调该过程以便同时重启少量节点。在重启更多节点之前，使用`nodetool status`验证重启的节点是否在线且正常，如果因为重启导致太多节点处于离线状态，群集可能会遇到暂时的服务降级或中断
2. 参考[替换ScyllaDB集群中的单个故障节点](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/replace-dead-node.html)替换当前故障节点

    集群中应该设置多个种子节点，但是不要将所有节点都设置为种子节点