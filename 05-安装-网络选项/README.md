# K3s安装 - 网络选项

默认情况下，K3s 将以 flannel 作为 CNI 运行，使用 VXLAN 作为默认后端。CNI和默认后端都可以通过参数修改。

## Flannel 选项

Flannel 的默认后端是 VXLAN。要启用加密，请使用下面的 IPSec（Internet Protocol Security）或 WireGuard 选项。

| CLI Flag 和 Value             | 描述                                                                    |
| :---------------------------- | :---------------------------------------------------------------------- |
| `--flannel-backend=vxlan`     | (默认) 使用 VXLAN 后端。                                                |
| `--flannel-backend=ipsec`     | 使用 IPSEC 后端，对网络流量进行加密。                                   |
| `--flannel-backend=host-gw`   | 使用 host-gw 后端。                                                     |
| `--flannel-backend=wireguard` | 使用 WireGuard 后端，对网络流量进行加密。可能需要额外的内核模块和配置。 |


#### flannel-backend 使用 `host-gw`

```
# K3s master
root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
        INSTALL_K3S_EXEC="--flannel-backend=host-gw" \
        INSTALL_K3S_MIRROR=cn sh -

# K3s agent 
root@k3s2:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
        INSTALL_K3S_MIRROR=cn K3S_URL=https://172.16.64.6:6443 \
        K3S_TOKEN=85892cbfef2177603f25be30344dbcd0 sh -
```

```
root@k3s1:~# cat /var/lib/rancher/k3s/agent/etc/flannel/net-conf.json
{
	"Network": "10.42.0.0/16",
	"Backend": {
	"Type": "host-gw"
}
}

root@k3s1:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.64.1     0.0.0.0         UG    100    0        0 enp0s2
10.42.0.0       0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.42.1.0       172.16.64.9     255.255.255.0   UG    0      0        0 enp0s2
172.16.64.0     0.0.0.0         255.255.255.0   U     0      0        0 enp0s2
172.16.64.1     0.0.0.0         255.255.255.255 UH    100    0        0 enp0s2
```

#### 启用 Directrouting

当主机在同一子网时，启用 direct routes(如host-gw)。`vxlan` 只用于将数据包封装到不同子网的主机上，同子网的主机之间使用 `host-gw`。默认值为 `false`。

```
# K3s master 和 agent
cat >> /etc/net-conf.json <<EOF 
{
        "Network": "10.42.0.0/16",
        "Backend": {
        "Type": "vxlan",
        "Directrouting": true
}
}
EOF

# K3s master
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
        INSTALL_K3S_EXEC="--flannel-conf=/etc/net-conf.json" \
        INSTALL_K3S_MIRROR=cn sh -
```

## 自定义 CNI

使用 `--flannel-backend=none` 运行 K3s，然后在安装你选择的 CNI。

#### Calico

按照[Calico CNI 插件指南](https://docs.projectcalico.org/master/reference/cni-plugin/configuration)。修改 Calico YAML，在 container_settings 部分中允许 IP 转发，例如：

```
"container_settings": {
              "allow_ip_forwarding": true
          }
```

> 如不配置 `"allow_ip_forwarding": true`， `svclb-traefik` 将会报错：`/usr/bin/entry: line 6: can't create /proc/sys/net/ipv4/ip_forward: Read-only file system`

```
root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | \
        INSTALL_K3S_MIRROR=cn \
        INSTALL_K3S_EXEC="--flannel-backend=none \
        --cluster-cidr=192.168.200.0/24" \
        sh -
root@k3s1:~# kubectl apply -f https://raw.githubusercontent.com/kingsd041/k3s-tutorial/main/05-安装-网络选项/calico.yaml
```

参考：https://docs.projectcalico.org/getting-started/kubernetes/k3s/quickstart

#### Cilium

参考：https://docs.cilium.io/en/v1.9/gettingstarted/k3s/


#### macvlan

支持


