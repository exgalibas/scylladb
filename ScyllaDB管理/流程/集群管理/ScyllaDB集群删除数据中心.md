## ScyllaDB集群删除数据中心

在从群集中删除（停用）数据中心之前，必须验证即将删除的数据中心是否包含任何唯一数据

### 前提

- 验证要删除的数据中心没有再接收客户端的请求

- 使用`nodetool status`命令检查集群状态以了解集群部署

    ```tex
    Datacenter: US-DC
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-lac8-23fddce9123e   B1
    UN  192.168.1.203  124.42 KB  256     32.6%             675ed9f4-6564-6dbd-can8-43fddce952gy   B1

    Datacenter: ASIA-DC
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  50.191.1.204  112.82 KB  256     32.7%             4d5ed9f4-7764-4ded-dad8-63fdace94b7c   B1
    UN  50.191.1.205  91.11 KB   256     32.9%             145id9f4-7777-1dvn-nac8-83fdzce917r4   B1
    UN  50.191.1.206  124.42 KB  256     32.6%             777ed9f4-6564-6dsd-can8-13fdxce999gy   B1

    Datacenter: EUROPE-DC
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  172.91.202.31  112.82 KB  256     32.7%             1d5ed8f4-7764-4dbd-rad8-44fddce94b7v   B1
    UN  172.91.202.32  91.11 KB   256     32.9%             525ed7g4-7437-1dbn-mac8-53fddce9123c   B1
    UN  172.91.202.33  124.42 KB  256     32.6%             975edbm4-6564-63bd-san8-73fddce952ga   B1
    ```

### 流程

1. 在要停用的数据中心的每个节点上运行命令`nodetool repair -pr`，检查将停用的数据中心与群集中的其他数据中心之间的所有数据是否同步

    比如：

    如果要删除ASIA-DC集群，则在ASIA-DC中的所有节点上运行`nodetool repair -pr`命令

2. 更改每个集群的keyspace，不再将数据复制到已停用的数据中心

    比如：

    ```shell
    cqlsh> DESCRIBE <KEYSPACE_NAME>
    cqlsh> CREATE KEYSPACE <KEYSPACE_NAME> WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', '<DC_NAME1>' : 3, '<DC_NAME2>' : 3, '<DC_NAME3>' : 3};

    cqlsh> ALTER KEYSPACE <KEYSPACE_NAME> WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', '<DC_NAME1>' : 3, '<DC_NAME2>' : 3};
    ```

    再比如：

    有三个数据中心：US-DC、ASIA-DC和EUROPE-DC，ASIA-DC将被移除，当前的keyspace是：

    ```shell
    cqlsh> DESCRIBE nba
    cqlsh> CREATE KEYSPACE nba WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'US-DC' : 3, 'ASIA-DC' : 2, 'EUROPE-DC' : 3};
    ```

    使用ALTER KEYSPACE更改KEYSPACE并移除已停用的数据中心

    ```shell
    cqlsh> ALTER KEYSPACE nba WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'US-DC' : 3, 'EUROPE-DC' : 3};
    ```

3. 在数据中心中要删除的每个节点上运行`nodetool decommission`，有关详细信息，请参阅[删除Scylla集群节点-缩容](https://opensource.docs.scylladb.com/stable/operating-scylla/procedures/cluster-management/remove-node.html)

    如果要删除ASIA-DC，请在此数据中心中的所有节点上执行`nodetool decommission`命令

4. 执行`nodetool status`验证是否已成功删除数据中心

    比如：

    ```tex
    Datacenter: US-DC
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  192.168.1.201  112.82 KB  256     32.7%             8d5ed9f4-7764-4dbd-bad8-43fddce94b7c   B1
    UN  192.168.1.202  91.11 KB   256     32.9%             125ed9f4-7777-1dbn-mac8-43fddce9123e   B1
    UN  192.168.1.203  124.42 KB  256     32.6%             675ed9f4-6564-6dbd-can8-43fddce952gy   B1

    Datacenter: EUROPE-DC
    Status=Up/Down
    State=Normal/Leaving/Joining/Moving
    --  Address        Load       Tokens  Owns (effective)                         Host ID         Rack
    UN  172.91.202.31  112.82 KB  256     32.7%             1d5ed8f4-7764-4dbd-rad8-44fddce94b7v   B1
    UN  172.91.202.32  91.11 KB   256     32.9%             525ed7g4-7437-1dbn-mac8-53fddce9123c   B1
    UN  172.91.202.33  124.42 KB  256     32.6%             975edbm4-6564-63bd-san8-73fddce952ga   B1
    ```