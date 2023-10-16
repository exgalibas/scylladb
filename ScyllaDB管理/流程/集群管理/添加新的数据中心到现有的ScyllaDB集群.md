## 添加新的数据中心到现有的ScyllaDB集群

以下过程指导如何将数据中心（DC）添加到运行中的Scylla集群中，可能是单个数据中心、多可用区或多数据中心。添加DC横向扩容集群并提供更高的可用性（HA）

流程包含：

- 在新DC上安装节点

- 将新节点逐个添加到群集中

- 更新所选keyspace的复制策略并使用新的DC

- 重建新节点

- 执行整个集群的修复（repair）

- 更新监控栈

注意：在开始从新数据中心读取数据之前，请务必完成上述整个过程。默认情况下，新添加的DC会用于数据读取

警告：您添加的一个或多个节点必须是干净的（无数据），否则您将面临数据丢失的风险，请参阅[清理节点数据](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/add-dc-to-existing-dc.html#clean-data-from-nodes)

### 前提

1. 登录到现有群集中的一个节点，从节点收集以下信息：

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

2. 在所有客户端应用程序上，将一致性级别切换为LOCAL_*（LOCAL_ONE、LOCAL_QUORUM等），以防止集群协调器访问新添加的数据中心

3. 在新DC安装新的干净的Scylla节点（请参阅[清理节点中的数据](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/add-dc-to-existing-dc.html#clean-data-from-nodes)），创建所需的任意数量的节点。按照Scylla安装过程操作直到scylla.yaml配置阶段，如果节点在安装过程中启动，请按[相关文档](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/clear-data.html)进行操作


### 清理节点中的数据

```shell
sudo rm -rf /var/lib/scylla/data
sudo find /var/lib/scylla/commitlog -type f -delete
sudo find /var/lib/scylla/hints -type f -delete
sudo find /var/lib/scylla/view_hints -type f -delete
```

### 添加新的DC

**流程**

警告：确保所有keyspace都使用NetworkTopologyStrategy，如果不是请按照[相关文档](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/update-topology-strategy-from-simple-to-network.html)进行操作

1. 对于现有数据中心中的每个节点，编辑scylla.yaml文件配置snitch：

     - Ec2MultiRegionSnitch - 适用于基于AWS云的多数据中心部署

     - GossipingPropertyFileSnitch - 适用于裸机和云（AWS 除外）部署

2. 设置DC和机架名称。在cassandra-rackdc.properties文件中编辑下面列出的参数，该文件可以在/etc/scylla/下找到

    如果使用`Ec2MultiRegionSnitch`

    Ec2MultiRegionSnitch为每个DC和机架提供默认名称，区域名称定义为数据中心名称，[可用区](https://opensource.docs.scylladb.com/stable/faq.html#faq-best-scenario-node-multi-availability-zone)定义为数据中心内的机架号

    比如：

    us-east-1区域中的节点，us-east是数据中心名称，1是机架号

    要更改DC名称，请执行以下操作：

    编辑cassandra-rackdc.properties文件，该文件可以在/etc/scylla/下找到

    文件中的配置项dc_suffix定义添加到数据中心名称的后缀，比如：

    对于us-east区域，dc_suffix=_1_scylla，那么数据中心名称就是us-east_1_scylla

    对于us-west区域，dc_suffix=_1_scylla，那么数据中心名称就是us-west_1_scylla

    如果使用`GossipingPropertyFileSnitch`

    - **dc** - 设置数据中心名称

    - **rack** - 设置机架名称


    比如：

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

3. 在现有数据中心中，逐个重新启动Scylla节点

    支持的操作系统

    ```shell
    sudo systemctl restart scylla-server
    ```

    Docker

    ```shell
    docker exec -it some-scylla supervisorctl restart scylla
    ```

4. 对于新数据中心中的每个节点，编辑下面列出的scylla.yaml文件参数，可以在/etc/scylla/下找到该文件

     - **cluster_name** - 设置统一集群名称

     - **seeds** - 设置为群集某个现有节点的IP地址，新节点将使用此IP连接到群集并了解群集拓扑和状态

     - **listen_address** - 监听其他Scylla节点连接的IP地址

     - **endpoint_snitch** - 设置snitch

     - **rpc_address** - 用于客户端连接（Thrift，CQL）的地址

     - **consistent_cluster_management** - 设置为与现有节点相同的值

     seeds、cluster_name和endpoint_snitch需要与现有集群匹配

5. 在新数据中心中，设置DC和机架名称（有关详细信息，请参阅步骤3）

6. 在新数据中心中，逐个启动Scylla节点

    支持的操作系统

    ```shell
    sudo systemctl start scylla-server
    ```

    Docker

    ```shell
    docker exec -it some-scylla supervisorctl start scylla
    ```

7. 使用`nodetool status`验证节点是否已添加到集群

    比如：

    ```tex
    $ nodetool status

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
    ```

8. 当所有节点启动并运行时，在新节点中更改以下keyspace
   - 用户创建的keyspace（需要复制到新DC）
   - 系统keyspace，比如system_auth, system_distributed和system_traces，将对应keyspace的数据复制到新DC中的三个节点

   例如：

   迁移之前

   ```
    DESCRIBE KEYSPACE mykeyspace;
    CREATE KEYSPACE mykeyspace WITH replication = { 'class' : 'NetworkTopologyStrategy', '<exiting_dc>' : 3};
   ```

   ALTER命令

   ```
    ALTER KEYSPACE mykeyspace WITH replication = { 'class' : 'NetworkTopologyStrategy', '<exiting_dc>' : 3, <new_dc> : 3};
    ALTER KEYSPACE system_auth WITH replication = { 'class' : 'NetworkTopologyStrategy', '<exiting_dc>' : 3, <new_dc> : 3};
    ALTER KEYSPACE system_distributed WITH replication = { 'class' : 'NetworkTopologyStrategy', '<exiting_dc>' : 3, <new_dc> : 3};
    ALTER KEYSPACE system_traces WITH replication = { 'class' : 'NetworkTopologyStrategy', '<exiting_dc>' : 3, <new_dc> : 3};
   ```

   迁移之后

   ```
    DESCRIBE KEYSPACE mykeyspace;
    CREATE KEYSPACE mykeyspace WITH REPLICATION = {'class’: 'NetworkTopologyStrategy', <exiting_dc>:3, <new_dc>: 3};
    CREATE KEYSPACE system_auth WITH replication = { 'class' : 'NetworkTopologyStrategy', '<exiting_dc>' : 3, <new_dc> : 3};
    CREATE KEYSPACE system_distributed WITH replication = { 'class' : 'NetworkTopologyStrategy', '<exiting_dc>' : 3, <new_dc> : 3};
    CREATE KEYSPACE system_traces WITH replication = { 'class' : 'NetworkTopologyStrategy', '<exiting_dc>' : 3, <new_dc> : 3};
   ```

9. 在新数据中心的每个节点中执行`nodetool rebuild`，并在重建命令中指定现有数据中心名称
    例如：

    ```
    nodetool rebuild <existing_data_center_name>
    ```
    重建可确保刚添加到群集的新节点能够识别新群集中的数据中心

10. 运行完整的集群修复，可以在每个节点上执行`nodetool repair -pr`，或使用 [Scylla管理器临时修复](https://manager.docs.scylladb.com/stable/repair)

11. 如果您之前使用了Scylla监控，请更新[监控栈](https://monitoring.docs.scylladb.com/stable/install/monitoring_stack.html#configure-scylla-nodes-from-files)以对新节点进行监控。如果您之前使用了Scylla管理器，请确保新节点安装[管理器代理](https://manager.docs.scylladb.com/stable/install-scylla-manager-agent.html)，并且管理器可以访问新的 DC

### 处理错误

如果其中一个新节点开始启动，但随后在中间过程中失败，例如由于断电，您可以重试（通过重新启动节点）。如果您不想重试，或者节点拒绝在后续尝试中启动，请参阅[处理成员更改失败文档](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/handling-membership-change-failures.html)

### 配置客户端不连接新DC

以下步骤将指导你如何配置你的客户端不连接新DC

以下示例适用于使用Java驱动程序的客户端，可对应修改此示例以满足您的实际需求

1. 防止客户端连接到新DC的一种方法是暂时限制它们仅使用与客户端位于同一DC中的节点。可以通过调整CL=LOCAL_*（例如LOCAL_QUORUM）和使用通过 DCAwareRoundRobinPolicy.Builder调用方法withLocalDc("dcLocalToTheClient")和withUsedHostsPerRemoteDc(0)创建的DCAwareRoundRobinPolicy来实现

    DCAwareRoundRobinPolicy的创建示例：

    ```
        variable = DCAwareRoundRobinPolicy.builder()
            .withLocalDc("<name of DC local to the client being configured>")
            .withUsedHostsPerRemoteDc(0)
            .build();
    ```
2. 新DC运行后，您可以从配置中删除"withUsedHostsPerRemoteDc(0)"，并将CL更改为以前的值

### Java客户端的其他资源

- [DCAwareRoundRobinPolicy.Builder](https://java-driver.docs.scylladb.com/scylla-3.10.2.x/api/com/datastax/driver/core/policies/DCAwareRoundRobinPolicy.Builder.html)
- [DCAwareRoundRobinPolicy](https://java-driver.docs.scylladb.com/scylla-3.10.2.x/api/com/datastax/driver/core/policies/DCAwareRoundRobinPolicy.html)