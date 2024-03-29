# 网络

本节课介绍了 CoreDNS、Traefik Ingress 控制器和 Klipper service load balancer 如何在 K3s 中工作。

关于 Flannel 配置选项和后端选择，或如何设置自己的 CNI，请参考[安装网络选项](http://docs.rancher.cn/docs/k3s/installation/network-options/_index)页面。

关于 K3s 需要开放哪些端口，请参考[安装要求](http://docs.rancher.cn/docs/k3s/installation/installation-requirements/_index#%E7%BD%91%E7%BB%9C)。

## CoreDNS

CoreDNS 是在 agent 节点启动时部署的。要禁用，请在每台服务器上运行 `--disable coredns` 选项。

如果你不安装 CoreDNS，你将需要自己安装一个集群 DNS 提供商。

#### 如何修改 coredns 参数

通过修改`/var/lib/rancher/k3s/server/manifests/coredns.yaml`配置文件会立即生效，但重启 K3s 服务会导致 coredns 配置重新初始化，所以要修改 corends 的参数，可以通过以下步骤修改：

1. 将`coredns.yaml`保存到其他目录
2. 通过 `--disable coredns` 禁用 coredns
3. 将备份的`coredns.yaml` 复制到 `/var/lib/rancher/k3s/server/manifests/c.yaml`，并修改对应参数

## Traefik Ingress Controller

启动 server 时，默认情况下会部署 Traefik。默认的配置文件在`/var/lib/rancher/k3s/server/manifests/traefik.yaml`中，对该文件的任何修改都会以类似`kubectl apply`的方式自动部署到 Kubernetes 中。

Traefik ingress controller 将使用主机上的 80 和 443 端口（即这些端口不能用于 HostPort 或 NodePort）。

Traefik 可以通过编辑`traefik.yaml`文件进行配置。为了防止 k3s 使用或覆盖修改后的版本，请使用`--disable traefik`部署 k3s，并将修改后的副本存储在`k3s/server/manifests`目录中。更多信息请参考官方的[Traefik 配置参数](https://github.com/helm/charts/tree/master/stable/traefik#configuration)。

要禁用它，请使用`--disable traefik`选项启动每个 server。

#### 如何启用 treafik2 dashboard：

```
# Note: in a kubernetes secret the string (e.g. generated by htpasswd) must be base64-encoded first.
# To create an encoded user:password pair, the following command can be used:
# htpasswd -nb admin admin | openssl base64

apiVersion: v1
kind: Secret
metadata:
  name: authsecret
  namespace: default
data:
  users: |2
    YWRtaW46JGFwcjEkLkUweHd1Z0EkUjBmLi85WndJNXZWRFMyR2F2LmtELwoK
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
spec:
  routes:
  - match: Host(`traefik.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))
    kind: Rule
    services:
    - name: api@internal
      kind: TraefikService
    middlewares:
      - name: auth
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: auth
spec:
  basicAuth:
    secret: authsecret # Kubernetes secret named "secretName"
```

然后通过 http://traefik.example.com/dashboard/ 访问 traefik dashboard

参考：

https://doc.traefik.io/traefik/operations/dashboard/

https://doc.traefik.io/traefik/middlewares/basicauth/#general

## Service Load Balancer

K3s 提供了一个名为[Klipper Load Balancer](https://github.com/rancher/klipper-lb)的负载均衡器，它可以使用可用的主机端口。 允许创建 LoadBalancer 类型的 Service，但不包括 LB 的实现。某些 LB 服务需要云提供商，例如 Amazon EC2 或 Microsoft Azure。相比之下，K3s service LB 使得可以在没有云提供商的情况下使用 LB 服务。

### Service LB 如何工作

K3s 创建了一个控制器，该控制器为 service load balancer 创建了一个 Pod，这个 Pod 是[Service](https://kubernetes.io/docs/concepts/services-networking/service/)类型的 Kubernetes 对象。

对于每个 service load balancer，都会创建一个[DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)。 DaemonSet 在每个节点上创建一个前缀为`svc`的 Pod。

Service LB 控制器会监听其他 Kubernetes Services。当它找到一个 Service 后，它会在所有节点上使用 DaemonSet 为该服务创建一个代理 Pod。这个 Pod 成为其他 Service 的代理，例如，来自节点上 8000 端口的请求可以被路由到端口 8888 上的工作负载。

如果创建多个 Services，则为每个 Service 创建一个单独的 DaemonSet。

只要使用不同的端口，就可以在同一节点上运行多个 Services。

如果您尝试创建一个在 80 端口上监听的 Service LB，Service LB 将尝试在集群中找到 80 端口的空闲主机。如果该端口没有可用的主机，LB 将保持 Pending 状态。

### 用法

在 K3s 中创建一个[LoadBalancer 类型的 Service](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer)。

### 从节点中排除 Service LB

要排除节点使用 Service LB，请将以下标签添加到不应排除的节点上：

```
svccontroller.k3s.cattle.io/enablelb
```

如果使用标签，则 service load balancer 仅在标记的节点上运行。

### 禁用 Service LB

要禁用嵌入式 LB，请使用`--disable servicelb`选项运行 k3s server。

如果您希望运行其他 LB，例如 MetalLB，这是必需的。

## 没有主机名的节点

一些云提供商（如 Linode）会以 "localhost "作为主机名创建机器，而其他提供商可能根本没有设置主机名。这可能会导致域名解析的问题。你可以用`--node-name`标志或`K3S_NODE_NAME`环境变量来运行 K3s，这样就会传递节点名称来解决这个问题。
