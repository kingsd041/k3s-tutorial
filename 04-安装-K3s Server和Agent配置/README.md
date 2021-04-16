# K3s Server/Agent 配置

- write-kubeconfig -- 将管理客户端的 kubeconfig 写入这个文件

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    K3S_KUBECONFIG_OUTPUT=/root/.kube/config \
    sh -

  root@k3s1:~# ls /root/.kube/config
  /root/.kube/config
  ```

- 使用 docker 作为容器运行时

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

- 针对多网卡主机安装 K3s 集群

  - K3s server:

    ```
    root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC="--node-ip=192.168.99.211" \
    sh -
    ```

  - K3s agent:
    ```
    root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    K3S_URL=https://192.168.99.211:6443 \
    K3S_TOKEN=9077b8e6f3b67b5f3e4a7723a96b199d \
    INSTALL_K3S_EXEC="--node-ip=192.168.99.212" \
    sh -
    ```

- --tls-san -- 在 TLS 证书中添加其他主机名或 IP 作为主题备用名称

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC="--tls-san 3.97.6.45"  \
    sh -
  ```

- 修改`kube-apiserver`、`kube-scheduler` 、`kube-controller-manager`、 `kube-cloud-controller-manager`、 `kubelet`、 `kube-proxy` 参数

  - kubelet-arg

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC='--kubelet-arg=max-pods=200' \
    sh -
  ```

  - kube-apiserver

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC='--kube-apiserver-arg=service-node-port-range=40000-50000' \
    sh -
  ```

  - kube-proxy-arg

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC='--kube-proxy-arg=proxy-mode=ipvs' \
    sh -
  ```

- --data-dir -- K3s 数据存储目录，默认为 `/var/lib/rancher/k3s` 或 `${HOME}/.rancher/k3s`(如果不是 root 用户)

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC='--data-dir=/opt/k3s-data' \
    sh -

  root@k3s1:~# ls /opt/k3s-data/
  agent  data  server
  ```

- --default-local-storage-path -- 本地存储类的默认存储路径

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC='--default-local-storage-path=/opt/storage' \
    sh -
  ```

- 禁用组件 --disable

  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC='--disable traefik' \
    sh -

  root@k3s1:~# ls /var/lib/rancher/k3s/server/manifests
  ccm.yaml  coredns.yaml  local-storage.yaml  metrics-server  rolebindings.yaml

  root@k3s1:~# kubectl get pods -A | grep traefik
  root@k3s1:~#
  ```

- 添加 label 和 taint
  ```
  root@k3s1:~# curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn \
    INSTALL_K3S_EXEC='--node-label foo=bar,hello=world --node-taint key1=value1:NoExecute' \
    sh -
  ```
