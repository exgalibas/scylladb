## ScyllaDB统一安装程序

本文档介绍如何使用Scylla统一安装程序进行安装、卸载和升级，当你没有服务器root权限时，建议使用统一安装程序。如果有root权限，建议下载对应操作系统的软件包(RPM和DEB)并使用软件包管理器(dnf和apt)安装它们

### 支持的发行版本

- CentOS 7 (仅支持root离线安装)

- CentOS 8

- Ubuntu 18.04 (如果NOFILE rlimit太低，则使用开发人员模式)

- Debian 10


### 下载和安装

对于没有root权限的安装，请按照[Scylla下载中心](https://www.scylladb.com/download/?platform=tar)上的说明进行操作

### 升级/降级/卸载

#### 升级

统一安装程序基于二进制包，由于不是RPM/DEB软件包，因此不会通过yum/apt升级或降级，目前只有scylla的install.sh支持升级

root：

```
./install.sh --upgrade
```

非root：

```
./install.sh --upgrade --nonroot
```

提示: 该操作不会升级scylla-jmx和scylla-tools

#### 卸载

root：

```
sudo ./uninstall.sh
```

非root：

```
./uninstall.sh --nonroot
```

#### 降级

先执行上述的卸载操作，再安装原始的Scylla软件包