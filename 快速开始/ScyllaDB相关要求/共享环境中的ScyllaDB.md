## 共享环境中的ScyllaDB

Scylla旨在利用机器上的所有资源，依赖磁盘和网络带宽、RAM和CPU，这使您能够以最少的节点数实现最大性能。但是，在开发和测试中，您的节点可能正在使用共享计算机，而Scylla无法控制该计算机并占用所有资源。本文介绍如何为共享环境配置Scylla，对于某些生产环境，这些设置值也可能是首选

请注意，Docker镜像是一个可行且更简单的选择 - [Scylla on dockerhub](https://hub.docker.com/r/scylladb/scylla/)

### 内存

Scylla消耗的最关键资源是内存。默认情况下，当Scylla启动时，它会检查节点的硬件配置并占用所有可用内存，除了为操作系统（OS）保留的一部分。这与大多数为操作系统保留大部分内存的开源数据库形成鲜明对比，但与大多数商业数据库相似

在共享环境中，特别是在台式机或笔记本电脑上，占用所有机器的可用内存会降低用户体验，因此Scylla允许将其内存使用量减少到给定的数量

在Ubuntu上，打开终端编辑`/etc/default/scylla-server`，添加`--memory 2G`将 Scylla 限制为2 GB RAM

在Red Hat/CentOS上，打开终端编辑`/etc/sysconfig/scylla-server`，添加`--memory 2G`将Scylla限制为2 GB RAM

如果从命令行启动Scylla，只需将`--memory 2G`附加到命令行即可

### CPU

默认情况下，Scylla将使用您的所有处理器（在某些配置中，特别是在Amazon AWS上，它可能会为操作系统留下一个内核）。此外Scylla会将其线程固定到特定内核，以最大限度地提高处理器芯片缓存的利用率。在专用节点上可以达到最大吞吐量，但在台式机或笔记本电脑上，它可能会导致用户界面缓慢

Scylla提供了两个选项来限制其CPU利用率：

- `--smp N`将Scylla限制为N个逻辑内核。例如使用`--smp 2`Scylla不会使用两个以上的逻辑内核

- `--overprovisioned`告诉Scylla它正在运行的机器也被其他进程使用，因此Scylla不会固定其线程或内存占用，并将轮询量减少到最低限度

在Ubuntu上，打开终端编辑`/etc/default/scylla-server`，添加`--smp 2 --overprovisioned`可以将Scylla限制到2个逻辑内核

在Red Hat/CentOS上，打开终端编辑`/etc/sysconfig/scylla-server`，添加`--smp 2 --overprovisioned`可以将Scylla限制到2个逻辑内核

如果从命令行启动Scylla，只需将`--smp 2 --overprovisioning`附加到命令行即可

### 其他限制

启动时，Scylla会检查硬件和操作系统配置，以验证它是否与Scylla的性能要求兼容。有关更多说明，请参阅[开发者模式](https://opensource.docs.scylladb.com/stable/getting-started/install-scylla/dev-mod.html)

### 概要

Scylla开箱即用，可立即用于生产，并具有极高性能。但针对开发或测试用途可能需要进行调整，简单的调整即可使Scylla服务器与其他进程或系统上的GUI共存