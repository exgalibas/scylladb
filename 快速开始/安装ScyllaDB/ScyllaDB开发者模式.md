## ScyllaDB开发者模式

如果你想在开发者模式下使用Scylla，你需要使用以下命令(使用root权限)

```shell
sudo scylla_dev_mode_setup --developer-mode 1
```

该命令会将开发者模式设置写入 `/etc/scylla.d/dev-mode.conf`