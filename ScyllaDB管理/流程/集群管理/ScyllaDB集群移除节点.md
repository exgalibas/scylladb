## ScyllaDB集群移除节点

可以从你的集群中移除部分节点来达到缩容的目的

### 移除正在运行的节点

1. 执行`nodetool status`检查集群中所有节点的状态

    ```shell
    Datacenter: DC1
        Status=Up/Down
        State=Normal/Leaving/Joining/Moving
        --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
        UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
        UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
        UN  192.168.1.203  124.42 KB  256     32.6%             675ed9f4-6564-6dbd-can8-43fddce952gy   B1
    ```

2. 如果节点状态为正常（UN），请登录/连接到要移除的节点并运行`nodetool decommission` 命令。建议使用`nodetool decommission`进行群集缩减操作，它可以将被移除节点的数据流式传输到集群中的其余节点来防止数据丢失

    如果节点的状态是`Joining`，移步到[安全删除正在加入的节点](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/safely-removing-joining-node.html)

    如果节点的状态是`Down`，移步到[移除不可用节点](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/remove-node.html#removing-an-unavailable-node)

    注意：需要注意剩余节点的剩余磁盘空间是否足够承接住被移除节点的数据，否则移除过程会失败，应该提前扩容剩余节点的磁盘空间

3. 运行`nodetool netstats`以监视令牌重新分配的进度

4. 运行`nodetool status`确保节点已被成功移除

    ```shell
    Datacenter: DC1
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
    ```

5. 人工清除被移除节点存储的数据和提交日志

    从群集中移除节点时，不会自动清除其数据，需要手动删除，可以使用以下命令删除数据：

    ```shell
    sudo rm -rf /var/lib/scylla/data
    sudo find /var/lib/scylla/commitlog -type f -delete
    sudo find /var/lib/scylla/hints -type f -delete
    sudo find /var/lib/scylla/view_hints -type f -delete
    ```


#### 处理错误

如果`nodetool decommission`在执行过程中失败了，比如因为机器掉电等，请参阅[处理成员更改失败文档](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/handling-membership-change-failures.html)

### 移除不可用节点

如果节点状态为正常关闭（DN），则应尝试恢复该节点。节点启动后，使用`nodetool decommission`（请参阅移除正在运行的节点）将其移除

如果所有恢复节点的尝试都失败并且节点已关闭，则可以通过运行`nodetool removenode` 并提供目标节点的主机ID来移除节点，有关详细信息，请参阅[nodetool removenode](https://opensource.docs.scylladb.com/stable/operating-scylla/nodetool-commands/removenode.html)

注意：

- `nodetool removenode`是最后的备用手段，只有当目标节点状态持续处于`Down`且无法恢复才可以使用

- 永远不要使用`nodetool removenode`移除正在运行且与其他节点通信正常的节点


Example：

```shell
nodetool removenode 675ed9f4-6564-6dbd-can8-43fddce952gy
```

`nodetool removenode`通知其他节点需要变更它拥有的令牌范围，并且节点应使用流式处理重新分发数据。如果数据源没有最新数据（如某个副本没有同步最新数据），则使用该命令不能保证重平衡后数据的一致性。此外，如果除目标节点的其他某些节点不可用或发生错误，`removenode`操作将失败。要确保操作成功并保持副本之间的一致性，您应该：

- 确保群集中所有其他节点的状态为正常（UN），如果一个或多个节点不可用，请参阅 [nodetool removenode](https://opensource.docs.scylladb.com/stable/operating-scylla/nodetool-commands/removenode.html)获取更多信息

- 运行`nodetool removenode`之前先在整个集群中执行一次repair，以保证所有的副本数据都同步达成一致

- 如果在`removenode`操作期间节点出现故障，请在重新运行`nodetool removenode`之前依然执行一次repair（如果启用了[RBNO](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/repair-based-node-operation.html)，则不需要执行repair）


### 其他信息

[Nodetool 相关](https://opensource.docs.scylladb.com/stable/operating-scylla/nodetool.html)