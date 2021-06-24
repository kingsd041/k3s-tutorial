# 仪表盘及卸载 K3S

## 仪表盘

- Kubernetes Dashboard
- Rancher UI
- kube-explorer

## 1. Kubernetes Dashboard

参考 [K3s 官网](http://docs.rancher.cn/docs/k3s/installation/kube-dashboard/_index/)

## 2. Rancher UI

可以将 K3s 导入到 Rancher UI 中去管理，参考 [Rancher 官网](http://docs.rancher.cn/docs/rancher2/cluster-provisioning/imported-clusters/_index/#%E5%AF%BC%E5%85%A5-k3s-%E9%9B%86%E7%BE%A4)

导入 K3s 集群时，Rancher 会将其识别为 K3s，除了其他导入的集群支持的功能之外，Rancher UI 还提供以下功能：

- 能够升级 K3s 版本。
- 能够配置在升级集群时，同时可以升级的最大节点数。
- 在主机详情页，能够查看（不能编辑）启动 K3s 集群时每个节点的 K3s 配置参数和环境变量。

## 3. kube-explorer

项目地址：https://github.com/cnrancher/kube-explorer

kube-explorer 是 Kubernetes 的便携式资源管理器，没有任何依赖。 并提供了一个几乎完全无状态的 Kubernetes 资源管理器。

#### 使用

从[发布页面](https://github.com/cnrancher/kube-explorer/releases)下载二进制文件

运行：

```
./kube-explorer --kubeconfig=xxxx --http-listen-port=9898 --https-listen-port=0
```

然后，打开浏览器访问 http://x.x.x.x:9898

# 卸载 K3S

要从 server 节点卸载 K3s，请运行：

```
/usr/local/bin/k3s-uninstall.sh
```

要从 agent 节点卸载 K3s，请运行：

```
/usr/local/bin/k3s-agent-uninstall.sh
```

#### 使用 docker 作为默认容器运行时

参考：Rancher [官网](http://docs.rancher.cn/docs/rancher2.5/cluster-admin/cleaning-cluster-nodes/_index/#%E6%B8%85%E7%90%86%E8%84%9A%E6%9C%AC)

```
#!/bin/bash

KUBE_SVC='
kubelet
kube-scheduler
kube-proxy
kube-controller-manager
kube-apiserver
'

for kube_svc in ${KUBE_SVC};
do
  # 停止服务
  if [[ `systemctl is-active ${kube_svc}` == 'active' ]]; then
    systemctl stop ${kube_svc}
  fi
  # 禁止服务开机启动
  if [[ `systemctl is-enabled ${kube_svc}` == 'enabled' ]]; then
    systemctl disable ${kube_svc}
  fi
done

# 停止所有容器
docker stop $(docker ps -aq)

# 删除所有容器
docker rm -f $(docker ps -qa)

# 删除所有容器卷
docker volume rm $(docker volume ls -q)

# 卸载mount目录
for mount in $(mount | grep tmpfs | grep '/var/lib/kubelet' | awk '{ print $3 }') /var/lib/kubelet /var/lib/rancher;
do
  umount $mount;
done

# 备份目录
mv /etc/kubernetes /etc/kubernetes-bak-$(date +"%Y%m%d%H%M")
mv /var/lib/etcd /var/lib/etcd-bak-$(date +"%Y%m%d%H%M")
mv /var/lib/rancher /var/lib/rancher-bak-$(date +"%Y%m%d%H%M")
mv /opt/rke /opt/rke-bak-$(date +"%Y%m%d%H%M")

# 删除残留路径
rm -rf /etc/ceph \
    /etc/cni \
    /opt/cni \
    /run/secrets/kubernetes.io \
    /run/calico \
    /run/flannel \
    /var/lib/calico \
    /var/lib/cni \
    /var/lib/kubelet \
    /var/log/containers \
    /var/log/kube-audit \
    /var/log/pods \
    /var/run/calico \
    /usr/libexec/kubernetes

# 清理网络接口
no_del_net_inter='
lo
docker0
eth
ens
bond
'

network_interface=`ls /sys/class/net`

for net_inter in $network_interface;
do
  if ! echo "${no_del_net_inter}" | grep -qE ${net_inter:0:3}; then
    ip link delete $net_inter
  fi
done

# 清理残留进程
port_list='
80
443
6443
2376
2379
2380
8472
9099
10250
10254
'

for port in $port_list;
do
  pid=`netstat -atlnup | grep $port | awk '{print $7}' | awk -F '/' '{print $1}' | grep -v - | sort -rnk2 | uniq`
  if [[ -n $pid ]]; then
    kill -9 $pid
  fi
done

kube_pid=`ps -ef | grep -v grep | grep kube | awk '{print $2}'`

if [[ -n $kube_pid ]]; then
  kill -9 $kube_pid
fi

# 清理Iptables表
## 注意：如果节点Iptables有特殊配置，以下命令请谨慎操作
sudo iptables --flush
sudo iptables --flush --table nat
sudo iptables --flush --table filter
sudo iptables --table nat --delete-chain
sudo iptables --table filter --delete-chain
systemctl restart docker
```
