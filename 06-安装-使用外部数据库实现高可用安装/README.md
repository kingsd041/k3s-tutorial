# K3s 安装 - 使用外部数据库实现高可用安装

本节介绍了如何使用外部数据库安装一个高可用的 K3s 集群。

## 要求

单节点 k3s server 集群可以满足各种用例，但是对于需要 Kubernetes control-plane 稳定运行的重要环境，您可以在 HA 配置中运行 K3s。一个 K3s HA 集群由以下几个部分组成：

- 两个或多个`server 节点`，将为 Kubernetes API 提供服务并运行其他 control-plane 服务。
- 零个或多个`agent 节点`，用于运行您的应用和服务。
- `外部数据存储` (与单个 k3s server 设置中使用的嵌入式 SQLite 数据存储相反)
- `固定的注册地址`，位于 server 节点的前面，以允许 agent 节点向集群注册

> Agent 通过固定的注册地址进行注册，但注册后直接与其中一个 server 节点建立连接。这是一个由 k3s agent 进程发起的 websocket 连接，并由作为 agent 进程一部分运行的客户端负载均衡器维护。

## 架构

![](http://docs.rancher.cn/assets/images/k3s-architecture-ha-server-46bf4c38e210246bda5920127bbecd53.png)

## 安装

0. 环境介绍
   主机名 | 角色 | IP
   -|-|-
   k3s-server-1 | k3s master | 172.31.2.134
   k3s-server-2 | k3s master | 172.31.2.42
   k3s-db | DB | 172.31.10.251
   k3s-lb | LB | 172.31.13.97
   k3s-agent | k3s agent | 172.31.15.130

1. 创建一个外部数据存储

你首先需要为集群创建一个外部数据存储。请参阅[集群数据存储选项](http://docs.rancher.cn/docs/k3s/installation/datastore/_index)文档了解更多细节。

K3s 支持以下数据存储选项：

- 嵌入式 SQLite
- PostgreSQL (经过认证的版本：10.7 和 11.5)
- MySQL (经过认证的版本：5.7)
- MariaDB (经过认证的版本：10.3.20)
- etcd (经过认证的版本：3.3.15)
- 嵌入式 etcd 高可用

```
# k3s-db
docker run --name some-mysql --restart=unless-stopped -p 3306:3306 -e MYSQL_ROOT_PASSWORD=password -d mysql:5.7
```

2. 启动 k3s server 节点

```
# k3s-server-1 和 k3s-server-2 节点
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="mysql://root:password@tcp(172.31.10.251:3306)/k3s" --tls-san 172.31.13.97
```

> --tls-san：在 TLS 证书中添加其他主机名或 IP 作为主题备用名称，本例为 LB 的 IP
> 否则通过 LB IP 连接 k3s api 时将会报错：`Unable to connect to the server: x509: certificate is valid for 10.43.0.1, 127.0.0.1, 172.31.2.134, 172.31.2.42, not 172.31.13.97`

```
root@k3s-server-1:~# kubectl get nodes
NAME           STATUS   ROLES                  AGE     VERSION
k3s-server-1   Ready    control-plane,master   7m15s   v1.20.7+k3s1
k3s-server-2   Ready    control-plane,master   3s      v1.20.7+k3s1
root@k3s-server-1:~# kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   local-path-provisioner-5ff76fc89d-jk4tx   1/1     Running     0          7m9s
kube-system   metrics-server-86cbb8457f-9vmq9           1/1     Running     0          7m9s
kube-system   coredns-854c77959c-b7hvc                  1/1     Running     0          7m9s
kube-system   helm-install-traefik-jvvrs                0/1     Completed   0          7m9s
kube-system   svclb-traefik-2pg2w                       2/2     Running     0          6m53s
kube-system   traefik-6f9cbd9bd4-lg8vl                  1/1     Running     0          6m53s
kube-system   svclb-traefik-qdnjz                       2/2     Running     0          7s
```

默认情况下，server 节点将是可调度的，因此你的工作负载可以在它们上启动。如果你希望有一个专用的 control-plane，在这个平面上不会运行用户工作负载，你可以使用 taints。`node-taint` 参数将允许你用污点配置节点，例如`--node-taint CriticalAddonsOnly=true:NoExecute`。

3. 配置固定的注册地址

Agent 节点需要一个 URL 来注册，你应该在 server 节点前面有一个稳定的 endpoint，不会随时间推移而改变。可以使用许多方法来设置此 endpoint，例如：

- 一个 4 层（TCP）负载均衡器
- 轮询 DNS
- 虚拟或弹性 IP 地址

使用 nginx 作为负载均衡器，将 6443 端口流量转发到 k3s server:

```
# k3s-lb 节点
cat >> /etc/nginx.conf <<EOF
worker_processes 4;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

stream {
    upstream k3s_api {
        least_conn;
        server 172.31.2.134:6443 max_fails=3 fail_timeout=5s;
        server 172.31.2.42:6443 max_fails=3 fail_timeout=5s;
    }
    server {
        listen     6443;
        proxy_pass k3s_api;
    }
}
EOF
```

```
docker run -d --restart=unless-stopped \
  -p 6443:6443 \
  -v /etc/nginx.conf:/etc/nginx/nginx.conf \
  nginx:1.14
```

4. 可选： 加入 Agent 节点

在 HA 集群中加入 agent 节点与在单个 server 集群中加入 agent 节点是一样的。你只需要指定 agent 应该注册到的 URL 和它应该使用的 token 即可。

```
# k3s-agent 节点
curl -sfL https://get.k3s.io | K3S_URL=https://172.31.13.97:6443 K3S_TOKEN=mynodetoken sh -

```

5. 通过 kubeconfig 访问 K3s 集群

此时，可以通过 LB IP 访问 k3s api:

```
kubectl get nodes
NAME           STATUS   ROLES                  AGE   VERSION
k3s-server-1   Ready    control-plane,master   68s   v1.20.7+k3s1
k3s-server-2   Ready    control-plane,master   66s   v1.20.7+k3s1
```

## 其他

如何在阿里云上搭建 K3s HA，可参考[这应该是最适合国内用户的 K3s HA 方案](https://mp.weixin.qq.com/s/0Wk2MzfWqMqt8DfUK_2ICA)
