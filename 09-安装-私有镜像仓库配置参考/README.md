# 私有镜像仓库配置参考

K3s 默认使用 containerd 作为容器运行时，所以在 docker 上配置镜像仓库是不生效的

K3s registry 配置目录为： `/etc/rancher/k3s/registries.yaml`。K3s 启动时，K3s 会检查 `/etc/rancher/k3s/` 中是否存在 `registries.yaml` 文件，并指示 containerd 使用文件中定义的镜像仓库。如果你想使用一个私有的镜像仓库，那么你需要在每个使用镜像仓库的节点上以 root 身份创建这个文件。

请注意，server 节点默认是可以调度的。如果你没有在 server 节点上设置污点，那么将在它们上运行工作负载，请确保在每个 server 节点上创建 `registries.yaml` 文件。

## 镜像仓库配置文件

K3s 镜像仓库配置文件由两大部分组成：`mirrors` 和 `configs`

- Mirrors 是一个用于定义专用镜像仓库的名称和 endpoint 的指令
- Configs 部分定义了每个 mirror 的 TLS 和证书配置。对于每个 mirror，你可以定义auth和/或tls

containerd 使用了类似 K8S 中 svc 与 endpoint 的概念，svc 可以理解为访问名称，这个名称会解析到对应的 endpoint 上。 也可以理解 mirror 配置就是一个反向代理，它把客户端的请求代理到 endpoint 配置的后端镜像仓库。mirror 名称可以随意填写，但是必须符合IP或域名的定义规则。并且可以配置多个 endpoint，默认解析到第一个 endpoint，如果第一个 endpoint 没有返回数据，则自动切换到第二个 endpoint，以此类推。

示例：
```
mirrors:
  "172.31.6.200:5000":
    endpoint:
      - "http://172.31.6.200:5000"
      - "http://x.x.x.x:5000"
      - "http://y.y.y.y:5000"
  "rancher.ksd.top:5000":
    endpoint:
      - "http://172.31.6.200:5000"
  "docker.io":
    endpoint:
      - "https://fogjl973.mirror.aliyuncs.com"
      - "https://registry-1.docker.io"

configs:
  "harbor2.kingsd.top":
    auth:
      username: admin
      password: Harbor@12345
    tls:
      cert_file: /home/ubuntu/harbor2.kingsd.top.cert
      key_file:  /home/ubuntu/harbor2.kingsd.top.key
      ca_file:   /home/ubuntu/ca.crt
```

可以通过 `crictl pull 172.31.6.200:5000/library/alpine` 和 `crictl pull rancher.ksd.top:5000/library/alpine` 获取到镜像，但镜像都是从同一个仓库获取到的。

```
root@rancher-server:/etc/rancher/k3s# systemctl restart k3s.service
root@rancher-server:/etc/rancher/k3s# crictl pull 172.31.6.200:5000/library/alpine
Image is up to date for sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e
root@rancher-server:/etc/rancher/k3s# crictl pull rancher.ksd.top:5000/library/alpine
Image is up to date for sha256:a24bb4013296f61e89ba57005a7b3e52274d8edd3ae2077d04395f806b63d83e
root@rancher-server:/etc/rancher/k3s#
```


## 使用 TLS 

#### 证书颁发机构颁发的证书

```
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "harbor.kingsd.top":
    endpoint:
      - "https://harbor.kingsd.top"
configs:
  "harbor.kingsd.top":
    auth:
      username: admin
      password: Harbor@12345
EOF
systemctl restart k3s
```

#### 自签名证书

```
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "harbor2.kingsd.top":
    endpoint:
      - "https://harbor2.kingsd.top"
configs:
  "harbor2.kingsd.top":
    auth:
      username: admin
      password: Harbor@12345
    tls:
      cert_file: /home/ubuntu/harbor2.kingsd.top.cert
      key_file:  /home/ubuntu/harbor2.kingsd.top.key
      ca_file:   /home/ubuntu/ca.crt
EOF
systemctl restart k3s
```

## 不使用 TLS (Http registry)

> 在没有 TLS 通信的情况下，需要为 endpoints 指定http://，否则将默认为 https

```
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "172.31.19.227:5000":
    endpoint:
      - "http://172.31.19.227:5000"
EOF
systemctl restart k3s
```

## 配置 Mirror

```
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "docker.io":
    endpoint:
      - "https://fogjl973.mirror.aliyuncs.com"
      - "https://registry-1.docker.io"
EOF
systemctl restart k3s
```

## 完整示例

```
mirrors:
  "harbor.kingsd.top":
    endpoint:
      - "https://harbor.kingsd.top"
  "harbor2.kingsd.top":
    endpoint:
      - "https://harbor2.kingsd.top"
  "172.31.19.227:5000":
    endpoint:
      - "http://172.31.19.227:5000"
  "docker.io":
    endpoint:
      - "https://fogjl973.mirror.aliyuncs.com"
      - "https://registry-1.docker.io"

configs:
  "harbor.kingsd.top":
    auth:
      username: admin
      password: Harbor@12345

  "harbor2.kingsd.top":
    auth:
      username: admin
      password: Harbor@12345
    tls:
      cert_file: /home/ubuntu/harbor2.kingsd.top.cert
      key_file:  /home/ubuntu/harbor2.kingsd.top.key
      ca_file:   /home/ubuntu/ca.crt

```

## 配置 Containerd

K3s 将会在`/var/lib/rancher/k3s/agent/etc/containerd/config.toml`中为 containerd 生成 `config.toml`。

如果要对这个文件进行高级设置，你可以在同一目录中创建另一个名为 `config.toml.tmpl` 的文件，此文件将会代替默认设置。

`config.toml.tmpl`将被视为 Go 模板文件，并且`config.Node`结构被传递给模板。[此模板](https://github.com/k3s-io/k3s/blob/master/pkg/agent/templates/templates.go#L16-L32)示例介绍了如何使用结构来自定义配置文件。