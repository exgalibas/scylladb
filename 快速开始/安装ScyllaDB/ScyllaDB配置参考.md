## ScyllaDB配置参考

本指南介绍可用于配置Scylla集群的命令，只要有对应目录的执行权限，即使没有sudo或root访问权限，也可以通过终端中的命令行运行这些命令（注意：始终只设置官方支持的配置项）

通过运行`scylla --help`可以列出所有Scylla命令的列表（提示：此命令显示所有Scylla命令和Seastar命令，Seastar命令被列为核心选项）

举个例子：

```
Scylla version 4.2.3-0.20210104.24346215c2 with build-id 0c8faf8bb8a3a0eda9337aad98ed3a6d814a4fa9 starting ...
command used: "scylla --help"
parsed command line options: [help]
Scylla options:
  -h [ --help ]                         show help message
  --version                             print version number and exit
  --options-file arg                    configuration file (i.e.
                                        <SCYLLA_HOME>/conf/scylla.yaml)
  --memtable-flush-static-shares arg    If set to higher than 0, ignore the
                                        controller's output and set the
                                        memtable shares statically. Do not set
                                        this unless you know what you are doing
                                        and suspect a problem in the
                                        controller. This option will be retired
                                        when the controller reaches more
                                        maturity
  --compaction-static-shares arg        If set to higher than 0, ignore the
                                        controller's output and set the
                                        compaction shares statically. Do not
                                        set this unless you know what you are
                                        doing and suspect a problem in the
                                        controller. This option will be retired
                                        when the controller reaches more
                                        maturity
```

### Scylla配置文件和命令

一些Scylla命令行命令源自`scylla.yaml`配置参数，比如`scylla.yaml`中的某个配置项`cluster_name: 'Test Cluster'`，通过运行

```shell
scylla --cluster-name 'Test Cluster'
```

可以修改该对应配置项的值，从上面的示例中可以看到，一般的经验法则是：

1. 从`scylla.yaml`中获取配置项

2. 在该配置项前添加`scylla --`

3. 将配置项中的下划线替换为短划线

4. 在终端运行命令