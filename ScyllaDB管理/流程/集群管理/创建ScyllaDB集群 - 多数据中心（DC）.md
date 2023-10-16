## 创建ScyllaDB集群 - 多数据中心（DC）

如果群集中的每个节点都有用于内部DC通信的内部IP和用于跨DC通信的外部IP，请参阅下表

### 第三方

多数据中心配置表

| 参数  | 多数据中心 |
| --- | --- |
| seeds | External IP address |
| listen_address | Internal IP address |
| rpc_address | Internal IP address |
| broadcast_address | External IP address |
| broadcast_rpc_address | External IP address |
| endpoint_snitch | GossipingPropertyFileSnitch |

**重点**

如果多数据中心的节点有两个物理网络接口：

- 将listen_address设置为此节点的私有IP或hostname

- 将broadcast_address设置为第二个IP或hostname（用于数据中心之间的通信）

- 将listen_on_broadcast_address设置为true

- 打开公共IP防火墙上的storage_port或ssl_storage_port


### 前提

- 确保所有相关[端口](https://opensource.docs.scylladb.com/stable/operating-scylla/admin.html#cqlsh-networking)都已打开

- 获取已为群集创建的所有节点的IP地址

- 为群集选择一个唯一的名称作为cluster_name（群集中的所有节点都相同）

- 选择要使用的snitch（群集中的所有节点都相同）。对于生产系统，建议使用DC-aware snitch，它可以支持keyspace使用NetworkTopologyStrategy[复制策略](https://opensource.docs.scylladb.com/stable/cql/ddl.html#create-keyspace-statement)

- 确定机架的名称，例如RACK1、RACK2或RC1、RC2

- 确定数据中心的名称，例如DC1、DC2或US-DC、ASIA-DC

**仔细选择数据中心名称，以后无法重命名数据中心**

使用生产环境时，必须选择以下snitch之一：

- [Ec2multiregionsnitch](https://opensource.docs.scylladb.com/stable/operating-scylla/system-configuration/snitch.html#snitch-ec2-multi-region-snitch) - 适用于AWS基于云的多数据中心部署

- [GossipingPropertyFileSnitch](https://opensource.docs.scylladb.com/stable/operating-scylla/system-configuration/snitch.html#snitch-gossiping-property-file-snitch) - 适用于裸机和云（AWS除外）部署


### 流程

需要为新群集中的每个节点执行这些步骤

1. 在节点上安装Scylla，具体请参阅上述相关文档，创建所需数量的节点。按照Scylla安装过程操作截止到scylla.yaml配置阶段，如果您的节点在此过程中启动，请按照[说明](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/clear-data.html)进行操作

2. 在scylla.yaml文件中，编辑下面列出的参数，该文件可以在`/etc/scylla/`下找到

     - **cluster_name** - 设置统一集群名称

     - **seeds** - 指定为第一个节点的IP，并且仅指定第一个节点。新节点将使用此种子节点的IP连接到集群并了解集群拓扑和状态

     - **listen_address** - 监听其他Scylla节点连接的IP地址

     - **endpoint_snitch** - 设置snitch

     - **rpc_address** - 用于客户端连接（Thrift，CQL）的地址

     - **consistent_cluster_management** - 默认情况下为 true，如果您不想在此集群中使用Raft进行一致的schema管理，则可以设置为 false（在后续更高版本中raft是必需的）。查看文档[ScyllaDB中的Raft](https://opensource.docs.scylladb.com/stable/architecture/raft.html)以了解更多信息

3. 在`cassandra-rackdc.properties`文件中，编辑机架和数据中心信息，该文件可以在 `/etc/scylla/`下找到

  若要节省带宽，请添加prefer_local=true参数，当节点位于同一数据中心时，Scylla将使用节点私有（本地）IP地址

4. 安装和配置Scylla并在所有节点上编辑scylla.yaml后，使用一个节点作为种子。启动种子节点，一旦它处于UN状态，就对所有其他节点重复此操作，每个节点都将处于UN状态

    支持的操作系统内

    ```shell
    sudo systemctl start scylla-server
    ```

    Docker

    ```shell
    docker exec -it some-scylla supervisorctl start scylla
    ```

5. 使用`nodetool status`命令验证节点是否已添加到群集中

6. 如果您运行的Scylla版本早于Scylla开源4.3或Scylla企业版2021.1，则需要更新每个节点的scylla.yaml文件中的seeds参数，以包含至少一个种子节点的IP。有关详细信息，请参阅[旧版本的Scylla](https://opensource.docs.scylladb.com/stable/kb/seed-nodes.html#seeds-older-versions)。应为每个DC指定至少两个种子节点，并使用公共IP


**例子：**

在此示例中，我们将展示如何安装包含九个节点的群集

1. 安装九个Scylla节点，每个数据中心三个节点（美国，亚洲，欧洲），对应IP是：

    ```tex
    U.S Data-center
    Node# Private IP    Public IP
    Node1 192.168.1.201 54.187.36.59 (seed)
    Node2 192.168.1.202 54.187.142.201
    Node3 192.168.1.203 54.187.168.20

    ASIA Data-center
    Node# Private IP    Public IP
    Node4 192.168.1.204 54.191.72.56
    Node5 192.168.1.205 54.187.25.99
    Node6 192.168.1.206 54.191.2.121

    EUROPE Data-center
    Node# Private IP    Public IP
    Node7 192.168.1.207 54.160.174.243
    Node8 192.168.1.208 54.235.9.159
    Node9 192.168.1.209 54.146.228.25
    ```

2. 在每个Scylla节点中，编辑scylla.yaml文件，相关操作请参阅上述

    **U.S Data-center - 192.168.1.201**

    ```yaml
    cluster_name: 'multi_dc_demo'
    seeds: "54.187.36.59"
    endpoint_snitch: GossipingPropertyFileSnitch
    rpc_address: "192.168.1.201"
    listen_address: "192.168.1.201"
    broadcast_address: "54.187.36.59"
    broadcast_rpc_address: "54.187.36.59"
    listen_on_broadcast_address: true (optional)
    ```

    **ASIA Data-center - 192.168.1.204**

    ```yaml
    cluster_name: 'multi_dc_demo'
    seeds: "54.187.36.59"
    endpoint_snitch: GossipingPropertyFileSnitch
    rpc_address: "192.168.1.204"
    listen_address: "192.168.1.204"
    broadcast_address: "54.191.72.56"
    broadcast_rpc_address: "54.191.72.56"
    listen_on_broadcast_address: true (optional)
    ```

    **EUROPE Data-center - 192.168.1.207**

    ```yaml
    cluster_name: 'multi_dc_demo'
    seeds: "54.187.36.59"
    endpoint_snitch: GossipingPropertyFileSnitch
    rpc_address: "192.168.1.207"
    listen_address: "192.168.1.207"
    broadcast_address: "54.160.174.243"
    broadcast_rpc_address: "54.160.174.243"
    listen_on_broadcast_address: true (optional)
    ```

3. 在每个Scylla节点中，编辑机架和数据中心信息相关的配置文件`cassandra-rackdc.properties`

    **Nodes 1-3**

    ```yaml
    dc=US-DC
    rack=RACK1
    ```

    **Nodes 4-6**

    ```yaml
    dc=ASIA-DC
    rack=RACK1
    ```

    **Nodes 7-9**

    ```yaml
    dc=EUROPE-DC
    rack=RACK1
    ```

4. 启动Scylla节点，从种子节点开始，等到它处于UN状态，然后对其他节点重复此操作

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
    nodetool status

    Datacenter: US-DC
    =========================
    Status=Up/Down
    |/ State=Normal/Leaving/Joining/Moving
    --   Address         Load            Tokens  Owns            Host ID                                 Rack
    UN   54.191.2.121    120.97 KB       256     ?               c84b80ea-cb60-422b-bc72-fa86ede4ac2e    RACK1
    UN   54.191.72.56    109.54 KB       256     ?               129087eb-9aea-4af6-92c6-99fdadb39c33    RACK1
    UN   54.187.25.99    104.94 KB       256     ?               0540c7d7-2622-4f1f-a3f0-acb39282e0fc    RACK1

    Datacenter: ASIA-DC
    =======================
    Status=Up/Down
    |/ State=Normal/Leaving/Joining/Moving
    --   Address         Load            Tokens  Owns            Host ID                                 Rack
    UN   54.160.174.243  109.54 KB       256     ?               c7686ffd-7a5b-4124-858e-df2e61130aaa    RACK1
    UN   54.235.9.159    109.75 KB       256     ?               39798227-9f6f-4868-8193-08570856c09a    RACK1
    UN   54.146.228.25   128.33 KB       256     ?               7a4957a1-9590-4434-9746-9c8a6f796a0c    RACK1

    Datacenter: EUROPE-DC
    =========================
    Status=Up/Down
    |/ State=Normal/Leaving/Joining/Moving
    --   Address         Load            Tokens  Owns            Host ID                                 Rack
    UN   54.187.36.59    114.35 KB       256     ?               4c3e1533-1b78-45bf-8bd4-818090f019ab    RACK1
    UN   54.187.142.201  109.54 KB       256     ?               d99967d6-987c-4a54-829d-86d1b921470f    RACK1
    UN   54.187.168.20   109.54 KB       256     ?               2329c2e0-64e1-41dc-8202-74403a40f851    RACK1
    ```