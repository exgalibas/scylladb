## 在EC2（单数据中心或多数据中心）上创建 ScyllaDB 集群

自ScyllaDB企业版2021.1.0和ScyllaDB开源4.5起，在EC2上运行Scylla集群的最简单方法是使用Scylla AMI，它是基于Ubuntu的。要使用不同的操作系统或您自己的AMI（Amazon机器镜像）或设置多DC Scylla集群，您需要自行配置Scylla集群，本页将指导您完成此过程

EC2上的Scylla集群可以部署为单DC集群或多DC集群，下表介绍了在单DC和多DC的场景下如何在scylla.yaml文件中为群集中的每个节点配置参数

有关Scylla AMI以及使用EC2用户数据配置scylla.yaml，请参阅[Scylla机器镜像](https://github.com/scylladb/scylla-machine-image)

最佳实践是将每个EC2区域用作Scylla DC，在这种情况下，节点使用区域内的内部（专用）IP进行通信，并使用外部（公共）IP进行区域（数据中心）之间通信

有关更多信息，请参阅AWS[实例寻址](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html)

### EC2配置表

| 参数  | 单DC | 多DC |
| --- | --- | --- |
| seeds | Internal IP address | External IP address |
| listen_address | Internal IP address | Internal IP address |
| rpc_address | Internal IP address | Internal IP address |
| broadcast_address | Internal IP address | External IP address |
| broadcast_rpc_address | Internal IP address | External IP address |
| endpoint_snitch | Ec2Snitch | Ec2MultiRegionSnitch |

### 前提

- 具有EC2实例的SSH访问权限

- 确保EC2安全组中所有相关[端口](https://opensource.docs.scylladb.com/stable/operating-scylla/admin.html#cqlsh-networking)都已打开

- 为群集选择一个唯一的名称作为cluster_name（群集中的所有节点都相同）


### 流程

对新群集中的每个节点执行以下步骤：

1. 在节点上安装Scylla，有关安装说明，请参阅上述文档，并按照该过程操作直至scylla.yaml配置阶段。如果Scylla服务已在运行（例如，如果您使用的是Scylla AMI），请根据[相关说明](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/clear-data.html)将其停止

2. 在位于`/etc/scylla/`中的scylla.yaml文件中编辑下面列出的参数。请参阅上面的EC2配置表，了解如何配置集群

     - **cluster_name** - 设置统一集群名称

     - **seeds** - 群集中第一个节点的IP地址，有关详细信息，请参阅[Scylla种子节点](https://opensource.docs.scylladb.com/stable/kb/seed-nodes.html)

     - **listen_address** - 监听其他Scylla节点连接的IP地址

     - **endpoint_snitch** - 设置snitch

     - **rpc_address** - 用于客户端连接（Thrift，CQL）的地址

     - **broadcast_address** - 被广播告知其他Scylla节点连接的地址

     - **broadcast_rpc_address** - 默认值未设置，要广播到其他Scylla节点的RPC地址。它不能设置为 0.0.0.0，如果留空，将使用rpc_address的值。如果rpc_address设置为 0.0.0.0，则必须显式配置broadcast_rpc_address

     - **consistent_cluster_management** - 默认情况下为true，如果您不想在此集群中使用Raft进行一致的schema管理，则可以设置为false（在后续更高版本中raft是必需的）。查看文档[ScyllaDB中的Raft](https://opensource.docs.scylladb.com/stable/architecture/raft.html)以了解更多信息

3. 在所有节点上安装和配置Scylla并编辑scylla.yaml文件后，启动seed节点，然后使用`sudo systemctl start scylla-server`顺序启动集群中的其余节点

4. 使用`nodetool status`命令验证节点是否已添加到群集中

### EC2snitch的默认DC和机架名称，以及如何覆盖DC名称

EC2snitch和Ec2MultiRegionSnitch为每个DC和机架提供默认名称，区域名称定义为数据中心名称，可用区定义为数据中心内的机架，无法更改机架名称

**例子**

对于us-east-1区域中的节点，us-east是数据中心名称，1是机架名称

若要更改数据中心的名称，请打开位于`/etc/scylla/`中的`cassandra-rackdc.properties`文件，然后编辑DC名称

dc_suffix定义添加到数据中心名称的后缀，例如：

- 对于区域us-east且后缀dc_suffix=_1_scylla，它完整的名称是us-east_1_scylla

- 对于区域us-west且后缀dc_suffix=_1_scylla，它完整的名称是us-west_1_scylla