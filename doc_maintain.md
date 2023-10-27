## 关于K8s的维护

### 1. node问题
#### 1.1 node重启
TODO
#### 1.2 node下线

### 2. 镜像管理
镜像存放位置取决于集群采用的容器运行时，首先配置crictl
```shell
# 若是docker作为运行时
crictl config runtime-endpoint unix:///var/run/cri-dockerd.sock

# 若是containerd作为运行时
crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```
>crictl 是一个与容器运行时接口 (Container Runtime Interface，CRI) 兼容的命令行工具。
> CRI 是 Kubernetes 用于与容器运行时 (container runtime) 交互的标准接口。它允许 Kubernetes 与不同的容器运行时，如 Docker、containerd、CRI-O 等，进行通信并管理容器的生命周期。

查看集群的【当前节点】使用过的镜像：
```shell
# 如果在master节点执行，那么一般看不到业务pod使用的镜像
$ crictl images                                                            
IMAGE                                                             TAG                 IMAGE ID            SIZE
docker.io/calico/cni                                              v3.26.1             9dee260ef7f59       93.4MB
docker.io/calico/kube-controllers                                 v3.26.1             1919f2787fa70       32.8MB
docker.io/calico/node                                             v3.26.1             8065b798a4d67       86.6MB
registry.aliyuncs.com/google_containers/coredns                   v1.9.3              5185b96f0becf       14.8MB
registry.aliyuncs.com/google_containers/etcd                      3.5.6-0             fce326961ae2d       103MB
registry.aliyuncs.com/google_containers/kube-apiserver            v1.25.14            48f6f02f2e904       35.1MB
registry.aliyuncs.com/google_containers/kube-controller-manager   v1.25.14            2fdc9124e4ab3       31.9MB
registry.aliyuncs.com/google_containers/kube-proxy                v1.25.14            b2d7e01cd611a       20.5MB
registry.aliyuncs.com/google_containers/kube-scheduler            v1.25.14            62a4b43588914       16.2MB
registry.aliyuncs.com/google_containers/pause                     3.8                 4873874c08efc       311kB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause         3.6                 6270bb605e12e       302kB
```

删除镜像
```shell
# 删除单个镜像
crictl rmi e6e09b2c69433
# 删除所有未使用到的镜像（用于释放磁盘空间）
crictl rmi --prune
```

也可以直接使用docker清理未使用的资源（如镜像，容器，卷，cache）：

```shell
# 查看可清理的资源
$ docker system df
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          1         0         4.904MB   4.904MB (100%)
Containers      0         0         0B        0B
Local Volumes   0         0         0B        0B
Build Cache     58        0         622.9MB   622.9MB

# 此命令可以用于清理磁盘，删除关闭的容器、无用的镜像和网络
# 添加 -f 禁用询问
docker system prune
```

### 2. Pod问题

### 2.1 Pod启动失败
现象：业务Pod一直处于`ContainerCreated`状态  
解决：describe pod的事件部分如下：
```shell
Events:
  Type     Reason                  Age                From               Message
  ----     ------                  ----               ----               -------
  Warning  FailedCreatePodSandBox  27h                kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to setup network for sandbox "8eb0d24a74f245b7cf603547f6b563984ea93396fd1640452c594f263bdd9ef0": plugin type="calico" failed (add): error getting ClusterInformation: connection is unauthorized: Unauthorized
  Normal   SandboxChanged          27h (x4 over 27h)  kubelet            Pod sandbox changed, it will be killed and re-created.
  Normal   Scheduled               46s                default-scheduler  Successfully assigned default/hellok8s-go-http-55cfd74847-w4bt4 to k8s-node1
```
查询：

- [K8S问题排查-升级K8S后apiserver的token超期问题](https://lyyao09.github.io/2023/05/14/k8s/K8S问题排查-升级K8S后apiserver的token超期问题/#more)
- [github issue](https://github.com/projectcalico/calico/issues/5712)

得知是calico pod使用的token过期，临时办法是delete calico pod，让它们重新创建，然后业务Pod就会正常了。

### 3. 清理资源

```shell
kubectl delete deployment,service --all
```

### 3. 使用Velero备份和恢复集群

介绍使用 [Velero](https://github.com/vmware-tanzu/velero/) 来完成（定期）备份集群和恢复集群。

使用前在其Github页面根据你的K8s版本选择Velero相应版本。

- [Velero工作原理](https://velero.io/docs/v1.12/how-velero-works/)

Velero支持按需备份、定时备份、恢复备份、设置备份过期等功能。
每个Velero操作如按需备份，计划备份，恢复都是一个自定义资源，使用Kubernetes定义 **自定义资源定义**（CRD）并存储在 etcd。Velero还包括处理自定义资源以执行备份、恢复和所有相关操作的控制器。

你可以备份或还原群集中的所有对象，也可以按类型、命名空间和/或标签筛选对象。

#### 3.1 备份

当执行`velero backup create test-backup`开始备份时，内部工作流如下：
- Velero客户端调用Kubernetes API服务器以创建Backup对象
- BackupController会注意到新的Backup对象并执行验证
- BackupController开始备份过程。它通过查询API服务器的资源来收集要备份的数据
- BackupController调用对象存储服务（例如AWS S3）以上传备份文件

#### 3.2 恢复

- **namespace重新映射**：恢复操作允许您从以前创建的备份中恢复所有对象和持久卷。您也可以只还原 对象和持久卷的过滤子集。Velero支持多个命名空间重新映射-例如，在单个恢复中，命名空间“abc”中的对象可以在命名空间“def”下重新创建，命名空间“123”中的对象可以在“456”下重新创建。
- **恢复的默认名称**：默认名称为<BACKUP NAME>-<TIMESTAMP>，其中<TIMESTAMP>格式为YYYYMMDDHhmmss。也可以指定自定义名称。恢复的对象还包括一个带有键velero.io/restore-name和值的标签<RESTORE NAME>；
- **还原**：默认情况下，备份存储位置以读写模式创建。但是，在还原过程中，您可以将备份存储位置配置为只读模式，这将禁用该存储位置的备份创建和删除。这有助于确保在还原方案中不会无意中创建或删除备份；
- **恢复钩子**：您可以选择指定 在恢复期间或恢复资源之后执行的恢复钩子。例如，您可能需要在数据库应用程序容器启动之前执行自定义数据库还原操作。


当执行`velero restore create`开始恢复时，内部工作流如下：
- Velero客户端调用Kubernetes API服务器以创建一个 还原对象；
- RestoreController会通知新的Restore对象并执行验证；
- RestoreController从对象存储服务获取备份信息。然后，它会对备份的资源运行一些预处理，以确保这些资源在新的集群上可以正常工作。例如使用 备份的API版本，以验证恢复资源是否可在目标群集上工作；
- RestoreController启动还原过程，一次还原一个符合条件的资源；

**非破坏性恢复**  
默认情况下，Velero执行非破坏性恢复，这意味着它不会删除目标群集上的任何数据。如果备份中的资源已存在于目标群集中，Velero将跳过该资源。您可以将Velero配置为使用更新策略，而不是使用 --existing-resource-policy还原标志。
当此标志设置为update时，Velero将尝试更新目标群集中的现有资源，以匹配备份中的资源。

#### 3.2 关于备份的API版本
Velero使用Kubernetes API服务器的首选版本为每个组/资源备份资源。恢复资源时，目标群集中必须存在相同的API组/版本，恢复才能成功。

例如，如果要备份的群集在things API组中有一个Gizmos资源，组/版本为things/v1 alpha 1、things/v1 beta1和things/v1，
并且服务器的首选组/版本为things/v1，则将从things/v1 API端点备份所有Gizmos。恢复此群集中的备份时，目标群集必须具有things/v1端点，
以便恢复Gizmo。注意，things/v1不需要是目标集群中的首选版本，它只需要存在。

#### 3.3 备份设置为过期

创建备份时，可以通过添加标志--ttl来指定TTL（生存时间）<DURATION>。如果Velero发现现有备份资源已过期，则会删除：
- 备份资源
- 云对象存储中的备份文件
- 所有PersistentVolume快照
- 所有关联的恢复

TTL标志允许用户使用以小时、分钟和秒为单位的值指定备份保留期，格式为--ttl 24h0m0s。如果未指定，则将应用默认TTL值30天。

过期策略不会立即应用，默认情况下，当gc控制器每小时运行一次协调循环时，会应用过期策略。
如果需要，您可以使用--garbage-collection-frequency标志调整协调循环的频率<DURATION>。

如果备份无法删除，则会将标签velero.io/gc-failure=<Reason>添加到备份自定义资源。您可以使用此标签筛选和选择未能删除的备份。
可能的原因有：
- 找不到备份存储位置
- BSL无法获取：无法从API服务器检索备份存储位置，原因不是找不到
- BSL只读：备份存储位置为只读

#### 3.4 对象存储使用方法

Velero可以将对象存储视为事实的来源。它会不断检查是否始终存在正确的备份资源。如果存储桶中有正确格式化的备份文件，但Kubernetes API中没有对应的备份资源，Velero会将对象存储中的信息重新存储到Kubernetes。

这个特性适用于恢复功能在群集迁移方案中工作，其中原始备份对象不存在于新群集中。

同样，如果一个已完成的备份对象存在于Kubernetes中，但不在对象存储中，它将从Kubernetes中删除，因为对象存储中的备份不存在。
对象存储同步不会删除失败或部分失败的备份。

#### 3.5 安装使用

```shell
wget --no-check-certificate https://hub.gitmirror.com/https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/

$ velero version                                                                                                                     
Client:
	Version: v1.12.0
	Git commit: 7112c62e493b0f7570f0e7cd2088f8cad968db99
<error getting server version: no matches for kind "ServerStatusRequest" in version "velero.io/v1">

```

Velero支持各种存储提供商，用于不同的备份和快照操作。Velero有一个插件系统，它允许任何人在不修改Velero代码库的情况下为其他备份和卷存储平台添加兼容性。

- [提供插件的存储提供商](https://velero.io/docs/v1.12/supported-providers/)

但现在不使用云提供商的存储资源，而是使用本地存储进行测试。