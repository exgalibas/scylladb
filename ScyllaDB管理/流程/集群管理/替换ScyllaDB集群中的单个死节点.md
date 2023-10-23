## 替换ScyllaDB集群中的单个死节点

替换死节点操作将导致群集中的其他节点将数据流式传输到被替换的节点，此操作可能需要一些时间（取决于数据大小和网络带宽）

此过程用于替换一个死节点。要替换多个死节点，请重复执行替换单死节点操作

### 前提

- 使用`nodetool status`验证节点的状态，状态为DN的节点需要更换

    ```shell
    Datacenter: DC1
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
    DN  192.168.1.203  124.42 KB  256     32.6%             675ed9f4-6564-6dbd-can8-43fddce952gy   B1
    ```
    注意：必须确保被替换的死节点永远不会重新加入到群集，因为这可能会导致脑裂。从集群网络或VPC中删除被替换的死节点
- 登录到死节点，如果有相应权限，请手动删除数据，使用以下命令删除数据：

    ```
    sudo rm -rf /var/lib/scylla/data
    sudo find /var/lib/scylla/commitlog -type f -delete
    sudo find /var/lib/scylla/hints -type f -delete
    sudo find /var/lib/scylla/view_hints -type f -delete
    ```
- 登录到群集中状态为UN的某个节点，并收集以下信息：
  - cluster_name - `cat /etc/scylla/scylla.yaml | grep cluster_name`
  - seeds - `cat /etc/scylla/scylla.yaml | grep seeds:`
  - endpoint_snitch - `cat /etc/scylla/scylla.yaml | grep endpoint_snitch`
  - Scylla version - `scylla --version`

### 流程
注意：如果您的Scylla版本早于Scylla开源4.3或Scylla企业版2021.1，请通过运行`cat /etc/scylla/scylla.yaml | grep seeds:`来检查死节点是否为种子节点：
  - 如果列出了死节点的IP地址，则死节点是种子节点，按照[替换死种子节点](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/replace-seed-node.html)中的说明替换种子节点
  - 如果未列出死节点的IP地址，则死节点不是种子节点，请继续按照以下步骤替换

同时建议检查所有节点的种子节点配置，有关详细信息请参阅[种子节点](https://opensource.docs.scylladb.com/stable/kb/seed-nodes.html)

1. 在新节点上安装Scylla，按照Scylla安装过程操作截止到scylla.yaml配置阶段，确保新节点安装的Scylla版本与当前集群中其他节点运行的Scylla版本一致
2. 在`scylla.yaml`文件中，编辑下面列出的参数，该文件可以在`/etc/scylla/`下找到：
    - cluster_name - 集群名
    - listen_address - 监听其他Scylla节点连接的IP地址
    - seeds - 种子节点
    - endpoint_snitch - 设置的snitch
    - rpc_address - 用于客户端连接（Thrift，CQL）的地址
    - consistent_cluster_management - 设置为现有其他节点使用的相同值
3. 将`replace_node_first_boot`参数添加到新节点的`scylla.yaml`配置文件中，成功替换节点后，也无需将其从`scylla.yaml`文件中删除（注意：不再支持过时的参数`replace_address`和`replace_address_first_boot`，不应使用），`replace_node_first_boot`参数的值应为要替换的死节点的主机ID

    例如：
    ```
    replace_node_first_boot: 675ed9f4-6564-6dbd-can8-43fddce952gy
    ```
4. 启动新节点的Scylla

    Supported OS
    ```
    sudo systemctl start scylla-server
    ```
    Docker
    ```
    docker exec -it some-scylla supervisorctl start scylla
    ```
5. 使用`nodetool status`确保新节点已加入到集群中

    例如：
    ```
    Datacenter: DC1
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
    DN  192.168.1.203  124.42 KB  256     32.6%             675ed9f4-6564-6dbd-can8-43fddce952gy   B1
    ```
    `192.168.1.203`是对应的死节点

    替换节点`192.168.1.204`将开始引导数据，在引导过程中，我们不会在节点状态列表中看到`192.168.1.204`

    使用`nodetool gossipinfo`查看到`192.168.1.204`处于`NORMAL`状态
    ```
    /192.168.1.204
    generation:1553759984
    heartbeat:104
    HOST_ID:655ae64d-e3fb-45cc-9792-2b648b151b67
    STATUS:NORMAL
    RELEASE_VERSION:3.0.8
    X3:3
    X5:
    NET_VERSION:0
    DC:DC1
    X4:0
    SCHEMA:2790c24e-39ff-3c0a-bf1c-cd61895b6ea1
    RPC_ADDRESS:192.168.1.204
    X2:
    RACK:B1
    INTERNAL_IP:192.168.1.204

    /192.168.1.203
    generation:1553759866
    heartbeat:2147483647
    HOST_ID:675ed9f4-6564-6dbd-can8-43fddce952gy
    STATUS:shutdown,true
    RELEASE_VERSION:3.0.8
    X3:3
    X5:0:18446744073709551615:1553759941343
    NET_VERSION:0
    DC:DC1
    X4:1
    SCHEMA:2790c24e-39ff-3c0a-bf1c-cd61895b6ea1
    RPC_ADDRESS:192.168.1.203
    RACK:B1
    LOAD:1.09776e+09
    INTERNAL_IP:192.168.1.203
    ```
    当新节点引导完成，`nodetool status`的节点列表中就会展示出新节点：
    ```
    Datacenter: DC1
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
    UN  192.168.1.204  124.42 KB  256     32.6%             655ae64d-e3fb-45cc-9792-2b648b151b67   B1
    ```
6. 在新节点中执行`nodetool repair`以确保该节点同步了其他节点的数据并一致，也可以通过[Scylla管理器](https://manager.docs.scylladb.com/)执行`repair`操作

#### 处理错误
如果新加入的节点在启动过程中发生了失败，比如发生掉电等，而且恢复后经过多次尝试仍然无法启动，请查阅[处理成员变更错误文档](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/handling-membership-change-failures.html)获取更多信息

### 重新启动后设置RAID
如果您需要重新启动（停止 + 启动，而不是reboot）具有临时存储的实例，例如EC2 i3或i3en节点，您应该注意：临时卷仅在实例的生存期内保留，当您停止、休眠或终止实例时，其实例存储卷中的应用程序和数据将被擦除（详情见https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/storage-optimized-instances.html）

在这种情况下，节点的数据将在重新启动后被清理，要解决此问题您需要重新创建RAID

1. 停止待重新启动的节点上的Scylla服务，以下命令也将在此节点上运行

    Supported OS
    ```
    docker exec -it some-scylla supervisorctl stop scylla
    ```
    Docker
    ```
    docker exec -it some-scylla supervisorctl stop scylla
    ```
2. 运行以下命令，记住在重新启动后不要挂载无效的RAID磁盘

    ```
    sudo sed -e '/.*scylla/s/^/#/g' -i /etc/fstab
    ```
3. 运行以下命令，将临时卷已擦除的实例（即要重新启动的节点的主机ID）替换为重新启动后的实例，重新启动的节点将被分配一个新的随机主机ID

    ```
    echo 'replace_node_first_boot: 675ed9f4-6564-6dbd-can8-43fddce952gy' | sudo tee --append /etc/scylla/scylla.yaml
    ```
4. 运行以下命令以重新设置RAID

    ```
    sudo /opt/scylladb/scylla-machine-image/scylla_create_devices
    ```
5. 启动Scylla服务

    Supported OS
    ```
    sudo systemctl stop scylla-server
    ```
    Docker
    ```
    docker exec -it some-scylla supervisorctl stop scylla
    ```
实例的公网/私网IP会在重启后可能发生变化，如果发生请参阅上面的替换过程