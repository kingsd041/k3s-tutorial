# 仪表盘及卸载 K3S

## 仪表盘

- Kubernetes Dashboard
- Rancher UI
- kube-explorer

### Kubernetes Dashboard

参考 [K3s 官网](http://docs.rancher.cn/docs/k3s/installation/kube-dashboard/_index/)

### kube-explorer

项目地址：https://github.com/cnrancher/kube-explorer

kube-explorer 是 Kubernetes 的便携式资源管理器，没有任何依赖。 并提供了一个几乎完全无状态的 Kubernetes 资源管理器。


#### 使用

从[发布页面](https://github.com/cnrancher/kube-explorer/releases)下载二进制文件

运行：

```
./kube-explorer --kubeconfig=xxxx --http-listen-port=9898 --https-listen-port=0
```

然后，打开浏览器访问 http://x.x.x.x:9898

### Rancher UI

可以将K3s导入到Rancher UI中去管理，参考 [Rancher 官网](http://docs.rancher.cn/docs/rancher2/cluster-provisioning/imported-clusters/_index/#%E5%AF%BC%E5%85%A5-k3s-%E9%9B%86%E7%BE%A4)

导入 K3s 集群时，Rancher 会将其识别为 K3s，除了其他导入的集群支持的功能之外，Rancher UI 还提供以下功能：

- 能够升级 K3s 版本。
- 能够配置在升级集群时，同时可以升级的最大节点数。
- 在主机详情页，能够查看（不能编辑）启动 K3s 集群时每个节点的 K3s 配置参数和环境变量。
