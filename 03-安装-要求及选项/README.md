# 安装要求及选项

## 安装要求

### 先决条件

- 两个节点不能有相同的主机名。

如果 k3s master 和 worker 节点主机名相同，注册 worker 节点时会报错：

```
Apr 13 07:37:37 k3s2 k3s[1763]: time="2021-04-13T15:37:37.444123090+08:00" level=error msg="Failed to retrieve agent config: Node password rejected, duplicate hostname or contents of '/etc/rancher/node/password' may not match server node-passwd entry, try enabling a unique node name with the --with-node-id flag"
```

- 如果您的所有节点都有相同的主机名，请使用`--with-node-id`选项为每个节点添加一个随机后缀，或者为您添加到集群的每个节点设计一个独特的名称，用`--node-name`或`$K3S_NODE_NAME`传递。

  - --with-node-id：

  ```
  curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.64.3:6443 K3S_TOKEN=6cc7b2022da90f9ee00256bdc2ee5926 sh -s - --with-node-id
  ```

  ```
  root@k3s1:~# kubectl get nodes
  NAME            STATUS   ROLES                  AGE     VERSION
  k3s1            Ready    control-plane,master   6m57s   v1.20.5+k3s1
  k3s1-a3c61eb3   Ready    <none                19s     v1.20.5+k3s1
  ```

  - 指定主机名：

  ```
  curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.64.3:6443 K3S_TOKEN=6cc7b2022da90f9ee00256bdc2ee5926 sh -s - --node-name k3s3
  ```

  ```
  curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | K3S_NODE_NAME="k3s3" INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.64.3:6443 K3S_TOKEN=6cc7b2022da90f9ee00256bdc2ee5926 sh -
  ```

  ```
  root@k3s1:~# kubectl get nodes
  NAME   STATUS     ROLES                  AGE   VERSION
  k3s1   Ready      control-plane,master   10m   v1.20.5+k3s1
  k3s3   Ready      <none                28s   v1.20.5+k3s1
  ```

### 操作系统

K3s 有望在大多数现代 Linux 系统上运行:

- CentOS: 7.8, 7.9, 8.2, 8.3
- RHEL: 7.8, 7.9, 8.2, 8.3
- SLES: 15SP2
- Ubuntu: 16.04, 18.04, 20.04

### 硬件

硬件要求根据您部署的规模而变化。这里列出了最低建议:

- 内存： 最低 512MB（建议至少为 1GB）
- CPU： 最低 1

#### 磁盘

K3s 的性能取决于数据库的性能。为了确保最佳速度，我们建议尽可能使用 SSD。在使用 SD 卡或 eMMC 的 ARM 设备上，磁盘性能会有所不同。

### 网络

**K3s Server 节点的入站规则：**

| 协议 | 端口      | 源                       | 描述                         |
| :--- | :-------- | :----------------------- | :--------------------------- |
| TCP  | 6443      | K3s agent 节点           | Kubernetes API Server        |
| UDP  | 8472      | K3s server 和 agent 节点 | 仅对 Flannel VXLAN 需要      |
| TCP  | 10250     | K3s server 和 agent 节点 | Kubelet metrics              |
| TCP  | 2379-2380 | K3s server 节点          | 只有嵌入式 etcd 高可用才需要 |

**通常情况下，所有出站流量都是允许的。**

## K3s 资源分析

### 影响资源利用率的因素

- K3s server：K3s server 的利用率数据主要是由支持 Kubernetes 数据存储（kine 或 etcd）、API Server、Controller-Manager 和 Scheduler 控制。创建/修改/删除资源将导致暂时的利用率上升。大量使用 Kubernetes 数据存储的 operators 或应用程序也将增加 server 的资源需求。

- K3s agent：管理镜像、提供存储或创建/销毁容器的操作将导致利用率的暂时上升，拉取镜像通常会影响 CPU 和 IO，因为它们涉及将镜像内容解压到磁盘。如果可能的话，工作负载存储(pod 临时存储和卷)应该与 agent 组件(/var/lib/rancher/k3s/agent)隔离，以确保不会出现资源冲突。

### 防止 agent 和工作负载干扰集群数据存储

在 server 节点运行工作负载 pod 时，应确保 agent 和工作负载 IOPS 不干扰数据存储。

将 server 组件（/var/lib/rancher/k3s/server）与 agent 组件（/var/lib/rancher/k3s/agent）放在不同的存储介质上，后者包括 containerd 镜像存储。

工作负载存储（pod 临时存储和卷）也应该与数据存储隔离。

## 安装选项

### 使用脚本安装的选项

- `INSTALL_K3S_SKIP_DOWNLOAD` -- (用于离线安装) 如果设置为 "true "将不会下载 K3s 的哈希值或二进制。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
      INSTALL_K3S_SKIP_DOWNLOAD=true \
      sh -
  [INFO]  Skipping k3s download and verify
  [ERROR]  Executable k3s binary not found at /usr/local/bin/k3s
  root@k3s1:~# wget http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  --2021-04-14 15:57:27--  http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  Resolving rancher-mirror.cnrancher.com (rancher-mirror.cnrancher.com)... 59.110.191.1
  Connecting to rancher-mirror.cnrancher.com (rancher-mirror.cnrancher.com)|59.110.191.1|:80... connected.
  HTTP request sent, awaiting response... 200 OK
  Length: 45387776 (43M) [application/octet-stream]
  Saving to: ‘k3s’

  k3s                                             100%[======================================================================================================>]  43.29M  4.61MB/s    in 11s

  2021-04-14 15:57:38 (4.09 MB/s) - ‘k3s’ saved [45387776/45387776]

  root@k3s1:~# chmod a+x k3s && mv k3s /usr/local/bin/
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
      INSTALL_K3S_SKIP_DOWNLOAD=true \
      sh -
  [INFO]  Skipping k3s download and verify
  [INFO]  Skipping installation of SELinux RPM
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Creating /usr/local/bin/ctr symlink to k3s
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s
  ```

- `INSTALL_K3S_SYMLINK` -- 默认情况下，如果路径中不存在命令，将为 kubectl、crictl 和 ctr 二进制文件创建符号链接。如果设置为'skip'将不会创建符号链接，而'force'将覆盖。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_SYMLINK=skip \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s
  ```

- `INSTALL_K3S_SKIP_ENABLE` -- 如果设置为 "true"，将不启用或启动 K3s 服务。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_SKIP_ENABLE=true \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Creating /usr/local/bin/ctr symlink to k3s
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service

  root@k3s1:~# systemctl enable /etc/systemd/system/k3s.service
  Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
  ```

- `INSTALL_K3S_SKIP_START` -- 如果设置为 "true "将不会启动 K3s 服务。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_SKIP_START=true \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Creating /usr/local/bin/ctr symlink to k3s
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit

  root@k3s1:~# systemctl start k3s.service
  ```

- `INSTALL_K3S_VERSION` -- 从 Github 下载 K3s 的版本。如果没有指定，将尝试从"stable"频道下载。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_VERSION="v1.19.9+k3s1" \
    sh -
  [INFO]  Using v1.19.9+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.19.9-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.19.9-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Creating /usr/local/bin/ctr symlink to k3s
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s
  ```

- `INSTALL_K3S_BIN_DIR` -- 安装 K3s 二进制文件、链接和卸载脚本的目录，或者使用/usr/local/bin 作为默认目录。
  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_BIN_DIR=/opt/bin \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /opt/bin/k3s
  [INFO]  Creating /opt/bin/kubectl symlink to k3s
  [INFO]  Creating /opt/bin/crictl symlink to k3s
  [INFO]  Creating /opt/bin/ctr symlink to k3s
  [INFO]  Creating killall script /opt/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /opt/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s
  root@k3s1:~# ls /opt/bin/
  crictl            ctr               k3s               k3s-killall.sh    k3s-uninstall.sh  kubectl
  ```

- `INSTALL_K3S_BIN_DIR_READ_ONLY` -- 如果设置为 true 将不会把文件写入INSTALL_K3S_BIN_DIR，强制设置INSTALL_K3S_SKIP_DOWNLOAD=true。`INSTALL_K3S_SKIP_DOWNLOAD` 会创建kubectl/crictl/ctr等，而 `INSTALL_K3S_BIN_DIR_READ_ONLY` 不创建。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_BIN_DIR_READ_ONLY=true \
    sh -
  [INFO]  Skipping k3s download and verify
  [ERROR]  Executable k3s binary not found at /usr/local/bin/k3s
  root@k3s1:~# cp k3s /usr/local/bin/
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_BIN_DIR_READ_ONLY=true \
    sh -
  [INFO]  Skipping k3s download and verify
  [INFO]  Skipping installation of SELinux RPM
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s

  root@k3s1:~# ls /usr/local/bin/
  k3s
  ```

- `INSTALL_K3S_SYSTEMD_DIR` -- 安装 systemd 服务和环境文件的目录，或者使用/etc/systemd/system 作为默认目录。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_SYSTEMD_DIR=/opt/systemd \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Creating /usr/local/bin/ctr symlink to k3s
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /opt/systemd/k3s.service.env
  [INFO]  systemd: Creating service file /opt/systemd/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s
  ```

- `INSTALL_K3S_EXEC` -- 带有标志的命令，用于在服务中启动 K3s。如果未指定命令，并且设置了 K3S_URL，它将默认为“agent”。如果未设置 K3S_URL，它将默认为“server”。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC="--docker" \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s
  ```

- `INSTALL_K3S_NAME` -- 要创建的 systemd 服务名称，如果以服务器方式运行 k3s，则默认为'k3s'；如果以 agent 方式运行 k3s，则默认为'k3s-agent'。如果指定了服务名，则服务名将以'k3s-'为前缀。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_NAME="demo" \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-demo-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s-demo.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s-demo.service
  [INFO]  systemd: Enabling k3s-demo unit
  [INFO]  systemd: Starting k3s-demo
  root@k3s1:~# systemctl status k3s-demo
  ```

- `INSTALL_K3S_TYPE` -- 要创建的 systemd 服务类型，默认为 notify

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_TYPE="exec" \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s

  root@k3s1:~# cat /etc/systemd/system/k3s.service
  [Unit]
  Description=Lightweight Kubernetes
  Documentation=https://k3s.io
  Wants=network-online.target
  After=network-online.target
  [Install]
  WantedBy=multi-user.target
  [Service]
  Type=exec
  EnvironmentFile=/etc/systemd/system/k3s.service.env
  KillMode=process
  Delegate=yes
  ```

- `INSTALL_K3S_SKIP_SELINUX_RPM` -- 如果设置为 "true "将跳过 k3s RPM 的自动安装。

  ```
  [root@ip-172-31-25-133 ~]# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_SKIP_SELINUX_RPM=true \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Skipping installation of SELinux RPM
  [ERROR]  Failed to find the k3s-selinux policy, please install:
      yum install -y container-selinux selinux-policy-base
      yum install -y https://rpm.rancher.io/k3s/stable/common/centos/7/noarch/k3s-selinux-0.2-1.el7_8.noarch.rpm
  ```

- `INSTALL_K3S_CHANNEL_URL` -- 用于获取 K3s 下载网址的频道 URL。默认为 https://update.k3s.io/v1-release/channels 。

- `INSTALL_K3S_CHANNEL` -- 用于获取 K3s 下载 URL 的通道。默认值为 "stable"。选项包括：`stable`, `latest`, `testing`。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_CHANNEL="latest" \
    sh -
  [INFO]  Finding release for channel latest
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s
  ```

- `K3S_CONFIG_FILE` -- 指定配置文件的位置。默认目录为`/etc/rancher/k3s/config.yaml`。

  ```
  root@k3s1:/opt# cat /opt/config.yaml
  node-label:
  - "foo=bar"
  - "something=amazing"

  root@k3s1:/opt# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    K3S_CONFIG_FILE=/opt/config.yaml \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s
  root@k3s1:/opt# kubectl get node --show-labels
  NAME   STATUS   ROLES                  AGE   VERSION        LABELS
  k3s1   Ready    control-plane,master   13s   v1.20.5+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,foo=bar,kubernetes.io/arch=amd64,kubernetes.io/hostname=k3s1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s,something=amazing
  ```

- `K3S_TOKEN` -- 用于将 server 或 agent 加入集群的共享 secret。

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    K3S_TOKEN=rancher-k3s \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s

  root@k3s1:~# cat /var/lib/rancher/k3s/server/token
  K1042465c14be8de6a57c482b4162f673addcb652acb13c8119a9900b5d27c234f7::server:rancher-k3s
  ```

- `K3S_TOKEN_FILE` -- 指定 `cluster-secret`,`token` 的文件目录。

  ```
  root@k3s1:~# cat /opt/token.txt
  rancher-k3s

  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    K3S_TOKEN_FILE=/opt/token.txt \
    sh -
  [INFO]  Finding release for channel stable
  [INFO]  Using v1.20.5+k3s1 as release
  [INFO]  Downloading hash http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/sha256sum-amd64.txt
  [INFO]  Downloading binary http://rancher-mirror.cnrancher.com/k3s/v1.20.5-k3s1/k3s
  [INFO]  Verifying binary download
  [INFO]  Installing k3s to /usr/local/bin/k3s
  [INFO]  Creating /usr/local/bin/kubectl symlink to k3s
  [INFO]  Creating /usr/local/bin/crictl symlink to k3s
  [INFO]  Skipping /usr/local/bin/ctr symlink to k3s, command exists in PATH at /usr/bin/ctr
  [INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
  [INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
  [INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
  [INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
  [INFO]  systemd: Enabling k3s unit
  [INFO]  systemd: Starting k3s

  root@k3s1:~# cat /var/lib/rancher/k3s/server/token
  K10d7a2e502103611961b4c6ab18baac06beb34f606886698c905f346671d6c432f::server:rancher-k3s
  ```

#### 其他说明

1. 以 "K3S\_"开头的环境变量将被保留，供 systemd 和 openrc 服务使用。
2. 在没有明确设置 exec 命令的情况下设置 K3S_URL，会将命令默认为 "agent"。
3. 运行 agent 时还必须设置 K3S_TOKEN。

### 从二进制中安装 K3s

安装脚本主要是配置 K3s 作为服务运行。如果你选择不使用脚本，你可以通过从我们的[发布页面](https://github.com/rancher/k3s/releases/latest)下载二进制文件，将其放在你的路径上，然后执行它来运行 K3s。K3s 二进制支持以下命令：

- `k3s server` -- 运行 K3s server，它还将启动 Kubernetes control-plane 组件，如 API server, controller-manager, 和 scheduler。

  ```
  root@k3s1:~# k3s server
  INFO[0000] Acquiring lock file /var/lib/rancher/k3s/data/.lock
  INFO[0000] Preparing data dir /var/lib/rancher/k3s/data/926943070e893920b703e893777d0cdc577dea7609f819abb14a97ea8ea1d30b
  INFO[2021-04-15T10:01:58.749717082+08:00] Starting k3s v1.20.5+k3s1 (355fff30)
  ...
  ...
  ...
  ```

- `k3s agent` -- 运行 K3s agent 节点。这将使 K3s 作为工作节点运行，启动 Kubernetes 节点服务 kubelet 和 kube-proxy。

  ```
  root@k3s2:~# k3s agent --server https://192.168.64.3:6443 --token 3c9c7aebc5cd83fb1ad92c5a7686eb95
  INFO[0000] Acquiring lock file /var/lib/rancher/k3s/data/.lock
  INFO[0000] Preparing data dir /var/lib/rancher/k3s/data/926943070e893920b703e893777d0cdc577dea7609f819abb14a97ea8ea1d30b
  INFO[2021-04-15T10:09:57.962476104+08:00] Starting k3s agent v1.20.5+k3s1 (355fff30)
  INFO[2021-04-15T10:09:58.351011343+08:00] Running load balancer 127.0.0.1:35105 -[192.168.64.3:6443]
  ...
  ...
  ...
  ```

- `k3s kubectl` -- 运行嵌入式 kubectl CLI。如果没有设置 KUBECONFIG 环境变量，当启动 K3s 服务器节点时，将自动尝试使用在/etc/rancher/k3s/k3s.yaml 创建的配置文件。

  ```
  root@k3s1:~# k3s kubectl get nodes
  NAME   STATUS   ROLES                  AGE     VERSION
  k3s1   Ready    control-plane,master   8m14s   v1.20.5+k3s1
  k3s2   Ready    <none                11s     v1.20.5+k3s1
  ```

- `k3s crictl` -- 运行一个嵌入式 crictl。这是一个用于与 Kubernetes 的容器运行时接口（CRI）交互的 CLI。对调试很有用。

  ```
  root@k3s1:~# k3s crictl ps
  CONTAINER           IMAGE               CREATED             STATE               NAME                     ATTEMPT             POD ID
  9ceb610df16c7       aa764f7db3051       8 minutes ago       Running             traefik                  0                   373c79416fa65
  cadceb62ae08d       897ce3c5fc8ff       8 minutes ago       Running             lb-port-443              0                   f8a0ecfe56562
  a26a49be485ac       897ce3c5fc8ff       8 minutes ago       Running             lb-port-80               0                   f8a0ecfe56562
  01894072f2298       148c192562719       8 minutes ago       Running             local-path-provisioner   1                   b9d55e63f632f
  5ccd6ed05120f       296a6d5035e2d       9 minutes ago       Running             coredns                  0                   2bae007d8e486
  be0765e77a703       9dd718864ce61       9 minutes ago       Running             metrics-server           0                   53ab949c026ce
  ```

- `k3s ctr` -- 运行一个嵌入式的 ctr。这是为 containerd（K3s 使用的容器守护进程）提供的 CLI。对调试很有用。
  ```
  root@k3s1:~# k3s ctr container ls
  CONTAINER                                                           IMAGE                                                                                                               RUNTIME
  01894072f2298208f3c109f9fb1d5e12e677d11cd5d0b0a3a66f550ae38644e4    docker.io/rancher/local-path-provisioner:v0.0.19                                                                    io.containerd.runc.v2
  2bae007d8e486afffbbf1ffb88e97b92d367aff4b06842217de4fb5d22ecf1b9    docker.io/rancher/pause:3.1
  ```

`k3s server` 和 `k3s agent` 命令有额外的配置选项，可以通过 `k3s server --help` 或 `k3s agent --help` 查看.

### 通过配置文件启动 K3s

除了使用环境变量和 CLI 参数来配置 K3s，K3s 还可以使用配置文件。默认目录位于/etc/rancher/k3s/config.yaml

    ```
    root@k3s1:~# cat /etc/rancher/k3s/config.yaml
    node-label:
    - "k3s-role=master"

    root@k3s1:~# k3s server
    INFO[2021-04-15T10:19:45.651637303+08:00] Starting k3s v1.20.5+k3s1 (355fff30)
    INFO[2021-04-15T10:19:45.652149355+08:00] Cluster bootstrap already complete
    INFO[2021-04-15T10:19:45.666066590+08:00] Configuring sqlite3 database connection pooling: maxIdleConns=2, maxOpenConns=0, connMaxLifetime=0s
    INFO[2021-04-15T10:19:45.666263783+08:00] Configuring database table schema and indexes, this may take a moment...
    ```

    ```
    root@k3s1:~# k3s kubectl get nodes --show-labels
    NAME   STATUS   ROLES                  AGE   VERSION        LABELS
    k3s1   Ready    control-plane,master   88s   v1.20.5+k3s1   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,beta.kubernetes.io/os=linux,k3s-role=master,k3s-rule=master,k3s.io/hostname=k3s1,k3s.io/internal-ip=192.168.64.3,kubernetes.io/arch=amd64,kubernetes.io/hostname=k3s1,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=true,node-role.kubernetes.io/master=true,node.kubernetes.io/instance-type=k3s
    ```

    如果同时使用配置文件和 CLI 参数。 在这种情况下，值将从两个来源加载，但 CLI 参数优先级更高。 对于可重复的参数，如--node-label，CLI 参数将覆盖列表中的所有值。
