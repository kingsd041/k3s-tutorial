# 离线安装

## 概述

离线安装的过程主要分为以下两个步骤：

**步骤 1**：部署镜像，本文提供了两种部署方式，分别是**部署私有镜像仓库**和**手动部署镜像**。请在这两种方式中选择一种执行。

**步骤 2**：安装 K3s，本文提供了两种安装方式，分别是**单节点安装**和**高可用安装**。完成镜像部署后，请在这两种方式中选择一种执行。

**离线升级 K3s 版本**：完成离线安装 K3s 后，您还可以通过脚本升级 K3s 版本，或启用自动升级功能，以保持离线环境中的 K3s 版本与最新的 K3s 版本同步。

## 通过私有镜像仓库安装K3s

#### 将所需镜像上传到私有镜像仓库

K3s 镜像列表可以从 https://github.com/k3s-io/k3s/releases 获取。

#### 创建镜像仓库 YAML

按照[私有镜像仓库配置指南](http://docs.rancher.cn/docs/k3s/installation/private-registry/_index/) 创建并配置`registry.yaml`文件。

```
mkdir -p /etc/rancher/k3s/
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "docker.io":
    endpoint:
      - "https://harbor.kingsd.top"
configs:
  "docker.io":
    auth:
      username: admin
      password: Harbor@12345
EOF
```

#### 安装单节点K3s


1. 从[K3s GitHub Release](https://github.com/rancher/k3s/releases)页面获取 K3s 二进制文件，K3s 二进制文件需要与离线镜像的版本匹配。

2. 获取 K3s 安装脚本：https://get.k3s.io。

3. 将二进制文件放在每个节点的`/usr/local/bin`中，并确保拥有可执行权限。将安装脚本放在每个节点的任意位置，并将其命名为`install.sh`。

4. 安装K3s server：
```
INSTALL_K3S_SKIP_DOWNLOAD=true ./install.sh
```

5. 将agent加入到K3s集群
```
INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken ./install.sh
```

## 通过手动部署镜像安装K3s

请按照以下步骤准备镜像和 K3s 二进制文件：

1. 从[K3s GitHub Release](https://github.com/rancher/k3s/releases)页面获取你所运行的 K3s 版本的镜像 tar 文件。

2. 将 tar 文件放在`images`目录下，例如：

```shell
sudo mkdir -p /var/lib/rancher/k3s/agent/images/
sudo cp ./k3s-airgap-images-$ARCH.tar /var/lib/rancher/k3s/agent/images/
```

3. 将 k3s 二进制文件放在 `/usr/local/bin/k3s`路径想，并确保拥有可执行权限。

4. 安装K3s server：
```
INSTALL_K3S_SKIP_DOWNLOAD=true ./install.sh
```

5. 将agent加入到K3s集群
```
INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://myserver:6443 K3S_TOKEN=mynodetoken ./install.sh
```

## 高可用安装

指定`INSTALL_K3S_SKIP_DOWNLOAD=true`参数指定使用本地K3s二进制文件进行安装。

```
INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_EXEC='server --datastore-endpoint=mysql://username:password@tcp(hostname:3306)/database-name' ./install.sh
```


## 升级 K3s

#### 通过脚本升级

离线环境的升级可以通过以下步骤完成：

1. 从[K3s GitHub Release](https://github.com/rancher/k3s/releases)页面下载要升级到的 K3s 版本。将 tar 文件放在每个节点的`/var/lib/rancher/k3s/agent/images/`目录下。删除旧的 tar 文件。
2. 复制并替换每个节点上`/usr/local/bin`中的旧 K3s 二进制文件。复制https://get.k3s.io 的安装脚本（因为它可能在上次发布后发生了变化）。再次运行脚本。
3. 重启 K3s 服务。
