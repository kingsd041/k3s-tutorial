# 嵌入式 DB 的高可用安装

要在这种模式下运行 K3s，你必须有奇数的服务器节点。我们建议从三个节点开始。

要开始运行，首先启动一个服务器节点，使用 cluster-init 标志来启用集群，并使用一个标记作为共享的密钥来加入其他服务器到集群中。

```
K3S_TOKEN=SECRET k3s server --cluster-init
或：
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - --cluster-init
```

启动第一台服务器后，使用共享密钥将第二台和第三台服务器加入集群。

```
K3S_TOKEN=SECRET k3s server --server https://<ip or hostname of server1>:6443。
或:
curl -sfL https://get.k3s.io | K3S_TOKEN=SECRET sh -s - --server https://<ip or hostname of server1>:6443
```

查询 ETCD 集群状态：

```
ETCDCTL_ENDPOINTS='https://172.31.12.136:2379,https://172.31.4.43:2379,https://172.31.4.190:2379' ETCDCTL_CACERT='/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt' ETCDCTL_CERT='/var/lib/rancher/k3s/server/tls/etcd/server-client.crt' ETCDCTL_KEY='/var/lib/rancher/k3s/server/tls/etcd/server-client.key' ETCDCTL_API=3 etcdctl endpoint status --write-out=table
```

> etcd 证书默认目录：/var/lib/rancher/k3s/server/tls/etcd
> etcd 数据默认目录：/var/lib/rancher/k3s/server/db/etcd
