## ScyllaDB日常管理程序以及如何禁用

始终建议运行最新版本的Scylla开源版本或Scylla企业版本。最新的稳定发布版本可从[下载中心](https://www.scylladb.com/download/)获得

安装Scylla时，默认情况下会安装两个服务：scylla-housekeeping-restart和scylla-housekeeping-daily。这些服务会检查最新的Scylla版本，并提示用户使用的版本是否早于公开可用的版本。有关您的Scylla部署信息，包括当前使用的Scylla版本，以及唯一的用户和服务器标识，均由集中式服务收集

如需禁用这些服务，可更改配置文件`/etc/scylla.d/housekeeping.cfg`，如下：

```
check-version: False
```