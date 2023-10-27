## ScyllaDB集群添加新节点 - 横向扩容

添加新节点时，群集中的其他节点会将数据流式传输到新节点，此操作称为引导，可能很耗时，具体取决于数据大小和网络带宽。如果使用[多可用区](https://opensource.docs.scylladb.com/stable/faq.html#faq-best-scenario-node-multi-availability-zone)，请确保它们是平衡的

### 前提

1. 在添加新节点之前，请使用`nodetool status`命令检查节点在集群中的状态，如果任何节点关闭，则无法向群集添加新节点

    比如：

    ```tex
    Datacenter: DC1
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    DN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
    ```

    在上面的示例中，IP地址为192.168.1.202的节点的状态为关闭（DN），如果要继续添加新节点，需要启动已关闭的节点或将其从群集中删除

2. 登录到群集中的一个节点以收集以下信息：

     - cluster_name - `grep cluster_name /etc/scylla/scylla.yaml`

     - seeds - `grep seeds: /etc/scylla/scylla.yaml`

     - endpoint_snitch - `grep endpoint_snitch /etc/scylla/scylla.yaml`

     - Scylla版本 - `scylla --version`

     - Authenticator - `grep authenticator /etc/scylla/scylla.yaml`

     - consistent_cluster_management - `grep consistent_cluster_management /etc/scylla/scylla.yaml`

注意：如果authenticator设置为PasswordAuthenticator，则应该增加system_auth keyspace的复制因子，比如：

```sql
ALTER KEYSPACE system_auth WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'dc1' : <new_replication_factor>}
```

建议将system_auth复制因子设置为每个DC中的节点数

### 流程

1. 在新节点上安装Scylla ，相关请参阅上述文档，请遵循Scylla安装过程直至scylla.yaml配置阶段，确保新节点的Scylla版本与群集中的其他节点相同

    如果节点在此过程中启动，请按照[节点自动启动时该怎么办](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/clear-data.html)

    注意：确保在新节点/替换节点上使用相同的Scylla补丁版本，以匹配集群的其余节点。不建议将具有不同版本的新节点添加到群集，例如使用以下命令安装Scylla补丁版本（实际应使用您已部署到其他节点的版本）

    - Scylla企业版 - `sudo yum install scylla-enterprise-2018.1.9`

    - Scylla开源 - `sudo yum install scylla-3.0.3`


    注意：在具有相同硬件的节点上保持I/O调度配置同步非常重要。这就是为什么我们建议添加与群集中现有节点完全相同的硬件设置的新节点时跳过执行scylla_io_setup（会设置`/etc/scylla.d/io_properties.yaml`文件，这个文件包含了I/O调优参数，用于优化Scylla数据库的性能），相反，我们建议在运行scylla_setup后将以下文件从现有节点复制到新节点，并重新启动scylla-server服务（如果它已在运行）：

    - `/etc/scylla.d/io.conf`

    - `/etc/scylla.d/io_properties.yaml`

    使用不同的I/O调度配置可能会导致不必要的瓶颈

2. 在/etc/scylla/中的scylla.yaml文件中，编辑以下参数：

     - **cluster_name** - 设置统一集群名称

     - **seeds** - 设置为群集某个现有节点的IP地址，新节点将使用此IP连接到群集并了解群集拓扑和状态

     - **listen_address** - 监听其他Scylla节点连接的IP地址

     - **endpoint_snitch** - 设置snitch

     - **rpc_address** - 用于客户端连接（Thrift，CQL）的地址

     - **consistent_cluster_management** - 设置为与现有节点相同的值

    注意：在早期版本的ScyllaDB中，种子节点有助于gossip。从Scylla开源4.3和Scylla企业版2021.1开始，gossip中的种子概念已被删除。如果您使用的是早期版本的ScyllaDB，则需要按以下方式配置seed参数：

    - 指定群集中当前种子节点的列表

    - 不要将要添加的节点设置为种子节点

    有关更多信息，请参阅[Scylla种子节点](https://opensource.docs.scylladb.com/stable/kb/seed-nodes.html)

    我们建议您将ScyllaDB更新到开源版本4.3及更高版本或企业版2021.1及更高版本

3. 使用以下命令启动ScyllaDB节点：

    支持的操作系统

    ```shell
    sudo systemctl start scylla-server
    ```

    Docker

    ```shell
    docker exec -it some-scylla supervisorctl start scylla
    ```

4. 使用`nodetool status`命令验证节点是否已添加到群集中。群集中的其他节点将数据流式传输到新节点，因此新节点将处于已启动且加入中（UJ）状态，最终节点的状态会更改为正常（UN），具体耗时取决于传输的数据大小和网络带宽

    例如：

    群集中的节点正在将数据流式传输到新节点：

    ```tex
    Datacenter: DC1
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
    UJ  192.168.1.203  124.42 KB  256     32.6%             675ed9f4-6564-6dbd-can8-43fddce952gy   B1
    ```

    群集中的节点已完成将数据流式传输到新节点：

    ```tex
    Datacenter: DC1
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
    UN  192.168.1.203  124.42 KB  256     32.6%             675ed9f4-6564-6dbd-can8-43fddce952gy   B1
    ```

5. 当新节点状态为正常（UN）时，请在群集中的所有节点上运行`nodetool cleanup`命令，除了刚添加的新节点，该命令会删除已传输到新添加的节点且不再由本节点拥有的数据

    注意：为了防止数据重复（冗余），必须在添加节点之后以及停用或删除任何节点之前完成清理。但是，清理可能会消耗大量资源，使用以下准则来减少清理影响：

    1. 添加多个节点时，请在所有节点上添加完成后，再在所有节点（最后添加的一个节点除外）运行清理操作

    2. 将清理操作推迟到业务量波谷期，同时确保在任何节点停用或删除之前成功完成清理

    3. 一次运行一个节点的清理，从而减少对群集的总体影响

6. 等到新节点在某个旧节点上运行的`nodetool status`输出中变为UN（正常）

    注意：如果您使用的是ScyllaDB开源4.3及更高版本或ScyllaDB企业版2021.1及更高版本，并且配置种子节点列表以参与gossip，您现在可以编辑scylla.yaml文件以将新节点添加到种子节点列表中。在scylla.yaml中修改种子列表后，无需重新启动Scylla服务

7. 如果您使用的是Scylla监控，请更新[监控栈](https://monitoring.docs.scylladb.com/stable/install/monitoring_stack.html#configure-scylla-nodes-from-files)以对其进行监控。如果您使用Scylla管理器，请确保新节点已安装[管理器代理](https://manager.docs.scylladb.com/stable/install-scylla-manager-agent.html)，保证管理器可以访问

### 处理失败

如果节点开始引导后期间意外失败，例如断电，您可以重新引导（通过重新启动节点）。如果您不想重试，或者节点在后续的尝试中拒绝引导，请参阅[处理集群成员更改失败文档](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/handling-membership-change-failures.html)