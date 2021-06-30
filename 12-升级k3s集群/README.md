# 升级 K3s 集群

当升级 K3s 时，K3s 服务会重启或停止，但 K3s 容器会继续运行。 要停止所有的 K3s 容器并重置容器的状态，可以使用 `k3s-killall.sh` 脚本。 killall 脚本清理容器、K3s 目录和网络组件，同时也删除了 iptables 链和所有相关规则。集群数据不会被删除。

## 手动升级

你可以通过使用安装脚本升级 K3s，或者手动安装所需版本的二进制文件。

> 注意： 升级时，先逐个升级 server 节点，然后再升级其他 agent 节点。

#### Channels 说明

通过安装脚本或使用我们的[自动升级](http://docs.rancher.cn/docs/k3s/upgrades/basic/_index)功能进行的升级可以绑定到不同的发布 channels。以下是可用的 channels。

| CHANNEL      | 描 述                                                                                                                                                  |
| :----------- | :----------------------------------------------------------------------------------------------------------------------------------------------------- |
| stable       | (默认)稳定版建议用于生产环境。这些版本已经过一段时间的社区强化。                                                                                       |
| latest       | 推荐使用最新版本尝试最新的功能。 这些版本还没有经过社区强化。                                                                                          |
| v1.19 (例子) | 每一个支持的 Kubernetes 次要版本都有一个发布 channel，它们分别是`v1.19`、`v1.20`和`v1.21`。。这些 channel 会选择最新的可用补丁版本，不一定是稳定版本。 |

对于详细的最新 channels 列表，您可以访问[k3s channel 服务 API](https://update.k3s.io/v1-release/channels)。关于 channels 工作的更多技术细节，请参见[channelserver 项目](https://github.com/rancher/channelserver)。

#### 使用安装脚本升级 K3s

要从旧版本升级 K3s，你可以使用**相同的标志**重新运行安装脚本:

- 升级到最新 stable 版本

```
curl -sfL https://get.k3s.io | sh -
```

- 升级到 latest 版本

```
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=latest sh -
```

- 升级到 v1.20 的最新版本

```
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL="v1.20" sh -
```

- 升级到指定版本

```
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=vX.Y.Z-rc1 sh -
```

#### 使用二进制文件手动升级 K3s

1. 从[发布](https://github.com/rancher/k3s/releases)下载所需版本的 K3s 二进制文件
2. 将下载的二进制文件复制到`/usr/local/bin/k3s`（或您所需的位置）
3. 停止旧的 K3s 二进制文件
4. 启动新的 K3s 二进制文件

## 自动升级

> 注意： 此功能从 v1.17.4+k3s1 开始提供支持。

你可以使用 Rancher 的 system-upgrad-controller 来管理 K3s 集群升级。这是一种 Kubernetes 原生的集群升级方法。它利用[自定义资源定义(CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#custom-resources)、`Job`和[控制器](https://kubernetes.io/docs/concepts/architecture/controller/)，根据配置的 Job 安排升级。

控制器通过监控 Job 和选择要在其上运行升级[ job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/) 的节点来调度升级。Job 通过[标签选择器](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)定义哪些节点应该升级。当一个 job 成功运行完成后，控制器会给它运行的节点打上相应的标签。

关于 system-upgrade-controller 的设计和架构或其与 K3s 集成的更多细节，请参见以下 Git 仓库：

- [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller)
- [k3s-upgrade](https://github.com/rancher/k3s-upgrade)

要以这种方式进行自动升级，你必须：

1. 将 system-upgrade-controller 安装到您的集群中
2. 配置 Job

## 安装 system-upgrade-controller

System-upgrade-controller 可以作为 deployment 安装到您的集群中。Deployment 需要一个 service-account、clusterRoleBinding 和一个 configmap。要安装这些组件，请运行以下命令:

```
kubectl apply -f https://github.com/rancher/system-upgrade-controller/releases/download/v0.6.2/system-upgrade-controller.yaml
```

#### 配置 Job

建议您最少创建两个 Job：升级 server（master）节点的 Job 和升级 agent（worker）节点的 Job。根据需要，您可以创建其他 Job 来控制跨节点的滚动升级。以下两个示例 Job 将把您的集群升级到 K3s v1.20.4+k3s1。创建 Job 后，控制器将接收这些 Job 并开始升级您的集群。

```
# Server plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: server-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: In
      values:
      - "true"
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.20.4+k3s1

---
# Agent plan
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: agent-plan
  namespace: system-upgrade
spec:
  concurrency: 1
  cordon: true
  nodeSelector:
    matchExpressions:
    - key: node-role.kubernetes.io/master
      operator: DoesNotExist
  prepare:
    args:
    - prepare
    - server-plan
    image: rancher/k3s-upgrade:v1.17.4-k3s1
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  version: v1.20.4+k3s1
```

关于这些 Job，有几个重要的事情需要提醒：

首先，必须在部署控制器的同一命名空间中创建 Job。

其次，`concurrency`字段表示可以同时升级多少个节点。

第三，`server-plan`通过指定一个标签选择器来选择带有`node-role.kubernetes.io/master`标签的节点，从而锁定 server 节点。`agent-plan`通过指定一个标签选择器来选择没有该标签的节点，以 agent 节点为目标。

第四，`agent-plan`中的 `prepare` 步骤会使该 Job 等待`server-plan`完成后再执行升级 jobs。

第五，两个 Job 的`version`字段都设置为 v1.17.4+k3s1。或者，你可以省略 `version` 字段，将 `channel` 字段设置为解析到 K3s 版本的 URL。这将导致控制器监控该 URL，并在它解析到新版本时随时升级集群。这与 [release channels](/docs/k3s/upgrades/basic/_index#发布-channels) 配合得很好。因此，你可以用下面的 channel 配置你的 Job，以确保你的集群总是自动升级到 K3s 的最新稳定版本。

```
apiVersion: upgrade.cattle.io/v1
kind: Plan
...
spec:
  ...
  channel: https://update.k3s.io/v1-release/channels/stable

```

如上所述，一旦控制器检测到 Job 已创建，升级就会立即开始。更新 Job 将使控制器重新评估 Job 并确定是否需要再次升级。

您可以通过 kubectl 查看 plans 和 jobs 来监控升级的进度：

```
kubectl -n system-upgrade get plans -o yaml
kubectl -n system-upgrade get jobs -o yaml
```
