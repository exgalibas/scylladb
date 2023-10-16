## <span id="scylladb_install">安装ScyllaDB</span>

ScyllaDB Web安装程序是一个与平台无关的安装脚本，可以通过curl将ScyllaDB安装在linux上也可以在[ScyllaDB安装中心](https://www.scylladb.com/download/#core)下载对应指定平台的安装包进行手动安装

### 前提

确保你的平台与想要安装的ScyllaDB版本兼容，具体请查阅[平台和版本与系统支持](https://opensource.docs.scylladb.com/stable/getting-started/os-support.html)

### 使用Web安装程序安装ScyllaDB

使用web安装程序安装ScyllaDB，执行命令行：

```
curl -sSf get.scylladb.com/server | sudo bash
```

安装脚本默认安装最新正式版本的开源ScyllaDB，可以使用以下选项选择不同的版本或者企业版：

| 选项  | 可接受的值 | 说明  |
| --- | --- | --- |
| --scylla-product | `scylla` \| `scylla-enterprise` | 指定ScyllaDB的类型：开源版本(scylla)或企业版(scylla-enterprise)，默认是scylla |
| --scylla-version | <版本号> | 指定ScyllaDB版本号，通过(x.y)指定主版本安装该主版本下最新的补丁版本，也可以通过(x.y.z)来指定发布的主版本和补丁版本，默认是最新的正式版本 |

可以使用 -h 或 --help 选项运行命令以打印有关脚本的帮助信息

### 例子

安装ScyllaDB开源版本4.6.1：

```
curl -sSf get.scylladb.com/server | sudo bash -s -- --scylla-version 4.6.1
```

安装ScyllaDB开源主版本4.6的最新补丁版本：

```
curl -sSf get.scylladb.com/server | sudo bash -s -- --scylla-version 4.6
```

安装ScyllaDB企业版2021.1：

```
curl -sSf get.scylladb.com/server | sudo bash -s -- --scylla-product scylla-enterprise --scylla-version 2021.1
```