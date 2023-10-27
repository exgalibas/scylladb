## ScyllaDB集群重新添加已下线节点
该篇主要介绍如何将已下线的节点重新添加到集群中。在某些情况下，可能有这样的诉求，比如因为误判/错误操作将某节点下线，以下步骤描述如何将其重新添加到集群中，包括清空数据以及作为一个新的节点加入到集群

### 前提
再次将节点加入到集群之前，需要通过`nodetool status`确认该节点不在集群中

比如：
已下线的节点IP是`192.168.1.203`，注意该节点不在集群中
```
Datacenter: DC1
  Status=Up/Down
  State=Normal/Leaving/Joining/Moving
  --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
  UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
  UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
```

### 步骤
1. 停止已下线节点的Scylla服务

    支持的操作系统
    ```
    sudo systemctl stop scylla-server
    ```
    Docker
    ```
    docker exec -it some-scylla supervisorctl stop scylla
    ```
2. 删除数据和日志文件

    ```
    sudo rm -rf /var/lib/scylla/data
    sudo find /var/lib/scylla/commitlog -type f -delete
    sudo find /var/lib/scylla/hints -type f -delete
    sudo find /var/lib/scylla/view_hints -type f -delete
    ```
    由于节点作为新节点添加回群集，因此必须删除节点的旧数据文件夹，否则节点旧的状态（如引导状态）将阻止其作为新节点启动初始化
3. 参考[将一个新节点添加到Scylla集群中](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/add-node-to-cluster.html)