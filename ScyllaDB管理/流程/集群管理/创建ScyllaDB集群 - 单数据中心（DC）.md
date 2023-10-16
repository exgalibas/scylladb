## 创建ScyllaDB集群 - 单数据中心（DC）

### 前提

- 确保所有相关[端口](https://opensource.docs.scylladb.com/stable/operating-scylla/admin.html#cqlsh-networking)都已打开

- 获取群集所有节点的IP地址

- 选择一个唯一的名称作为群集的cluster_name（群集中的所有节点的cluster_name都相同）

- 选择要使用的[snitch](https://opensource.docs.scylladb.com/stable/faq.html#faq-snitch-strategy)（群集中的所有节点都相同），对于生产系统，建议使用DC-aware snitch，它可以支持keyspace使用NetworkTopologyStrategy[复制策略](https://opensource.docs.scylladb.com/stable/cql/ddl.html#create-keyspace-statement)


### 流程

需要为新群集中的每个节点执行这些步骤

1. 在节点上安装Scylla。按照Scylla安装过程操作截止到scylla.yaml配置阶段，如果节点在此过程中启动，请按照[说明](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/clear-data.html)进行操作

2. 在scylla.yaml文件中，编辑下面列出的参数，该文件可以在`/etc/scylla/`下找到

     - **cluster_name** - 设置统一集群名称

     - **seeds** - 指定为第一个节点的IP，并且仅指定第一个节点。新节点将使用此种子节点的IP连接到集群并了解集群拓扑和状态

     - **listen_address** - 监听其他Scylla节点连接的IP地址

     - **endpoint_snitch** - 设置snitch

     - **rpc_address** - 用于客户端连接（Thrift，CQL）的地址

     - **consistent_cluster_management** - 默认情况下为true，如果您不想在此集群中使用Raft进行一致的schema管理，则可以设置为false（在后续更高版本中raft是必需的）。查看文档[ScyllaDB中的Raft](https://opensource.docs.scylladb.com/stable/architecture/raft.html)以了解更多信息

3. 仅当您使用GossipingPropertyFileSnitch时，才需要执行此步骤，如果没有，请跳过此步骤。在`cassandra-rackdc.properties`文件中，编辑下面列出的参数，该文件可以在`/etc/scylla/`下找到

    例子：

    ```yaml
    # cassandra-rackdc.properties
    #
    # The lines may include white spaces at the beginning and the end.
    # The rack and data center names may also include white spaces.
    # All trailing and leading white spaces will be trimmed.
    #
    dc=thedatacentername
    rack=therackname
    # prefer_local=<false | true>
    # dc_suffix=<Data Center name suffix, used by EC2SnitchXXX snitches>
    ```

4. 安装和配置Scylla后，使用第一个节点作为种子节点，在所有其他节点上编辑scylla.yaml 文件。启动种子节点，一旦它处于UN状态，就对所有其他节点重复此操作，每个节点都将处于UN状态

    支持的操作系统内

    ```shell
    sudo systemctl start scylla-server
    ```

    Docker

    ```shell
    docker exec -it some-scylla supervisorctl start scylla
    ```

5. 使用`nodetool status`验证节点是否已添加到群集

6. 如果您运行的Scylla版本早于Scylla开源4.3或Scylla企业版2021.1，则需要更新每个节点的scylla.yaml文件中的seeds参数，以包含至少一个种子节点的IP。有关详细信息，请参阅[旧版本的Scylla](https://opensource.docs.scylladb.com/stable/kb/seed-nodes.html#seeds-older-versions)


### 例子

此示例说明如何使用GossipingPropertyFileSnitch作为endpoint_snitch安装和配置三节点集群，每个节点位于不同的机架上

1. 安装三个Scylla节点，IP 是：

    ```tex
    192.168.1.201 (seed)
    192.168.1.202
    192.168.1.203
    ```

2. 在每个Scylla节点中，编辑scylla.yaml文件

    **192.168.1.201**

    ```yaml
    cluster_name: 'Scylla_cluster_demo'
    seeds: "192.168.1.201"
    endpoint_snitch: GossipingPropertyFileSnitch
    rpc_address: "192.168.1.201"
    listen_address: "192.168.1.201"
    ```

    **192.168.1.202**

    ```yaml
    cluster_name: 'Scylla_cluster_demo'
    seeds: "192.168.1.201"
    endpoint_snitch: GossipingPropertyFileSnitch
    rpc_address: "192.168.1.202"
    listen_address: "192.168.1.202"
    ```

    **192.168.1.203**

    ```yaml
    cluster_name: 'Scylla_cluster_demo'
    seeds: "192.168.1.201"
    endpoint_snitch: GossipingPropertyFileSnitch
    rpc_address: "192.168.1.203"
    listen_address: "192.168.1.203"
    ```

3. 仅当使用GossipingPropertyFileSnitch时才需要执行此步骤，在每个Scylla节点中，编辑cassandra-rackdc.properties文件

    **192.168.1.201**

    ```yaml
    # cassandra-rackdc.properties
    #
    # The lines may include white spaces at the beginning and the end.
    # The rack and data center names may also include white spaces.
    # All trailing and leading white spaces will be trimmed.
    #
    dc=datacenter1
    rack=rack43
    # prefer_local=<false | true>
    # dc_suffix=<Data Center name suffix, used by EC2SnitchXXX snitches>
    ```

    **192.168.1.202**

    ```yaml
    # cassandra-rackdc.properties
    #
    # The lines may include white spaces at the beginning and the end.
    # The rack and data center names may also include white spaces.
    # All trailing and leading white spaces will be trimmed.
    #
    dc=datacenter1
    rack=rack44
    # prefer_local=<false | true>
    # dc_suffix=<Data Center name suffix, used by EC2SnitchXXX snitches>
    ```

    **192.168.1.203**

    ```yaml
    # cassandra-rackdc.properties
    #
    # The lines may include white spaces at the beginning and the end.
    # The rack and data center names may also include white spaces.
    # All trailing and leading white spaces will be trimmed.
    #
    dc=datacenter1
    rack=rack45
    # prefer_local=<false | true>
    # dc_suffix=<Data Center name suffix, used by EC2SnitchXXX snitches>
    ```

4. 启动Scylla节点，由于我们的种子节点是`192.168.1.201`，我们将首先启动它，等到它处于UN状态，然后再启动其他节点

    支持的操作系统内

    ```shell
    sudo systemctl start scylla-server
    ```

    Docker

    ```shell
    docker exec -it some-scylla supervisorctl start scylla
    ```

5. 使用`nodetool status`命令验证节点是否已添加到群集中

    ```shell
    Datacenter: datacenter1
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   43
    UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   44
    UN  192.168.1.203  124.42 KB  256     32.6%             675ed9f4-6564-6dbd-can8-43fddce952gy   45
    ```