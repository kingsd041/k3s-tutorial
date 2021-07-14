# 卷和存储

当部署一个需要保留数据的应用程序时，你需要创建持久存储。持久存储允许您从运行应用程序的 pod 外部存储应用程序数据。即使应用程序的 pod 发生故障，这种存储方式也可以使您维护应用程序数据。

本节介绍了如何通过 local storage provider 或 Longhorn 来设置持久存储。

## 设置 Local Storage Provider

K3s 自带 Rancher 的 Local Path Provisioner，这使得能够使用各自节点上的本地存储来开箱即用地创建持久卷声明。

Local Path Provisioner 为 Kubernetes/K3s 用户提供了一种利用每个节点中的本地存储的方法。根据用户配置，Local Path Provisioner 将自动在节点上创建基于 hostPath 的持久卷。它利用了 Kubernetes Local Persistent Volume 特性引入的特性，但他比 Kubernetes 中内置的 `local` 卷特性更简单的解决方案。

**pvc.yaml**

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-path-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 2Gi
```

**pod.yaml**

```
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
  namespace: default
spec:
  containers:
  - name: volume-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: volv
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: volv
    persistentVolumeClaim:
      claimName: local-path-pvc
```

**应用 yaml:**

```
kubectl create -f pvc.yaml
kubectl create -f pod.yaml
```

**确认 PV 和 PVC 已创建：**

```
kubectl get pv
kubectl get pvc
```

## 设置 Longhorn

K3s 支持 [Longhorn](https://github.com/longhorn/longhorn). Longhorn 是 Kubernetes 的一个开源分布式块存储系统。

下面我们介绍一个简单的例子。有关更多信息，请参阅[官方文档](https://github.com/longhorn/longhorn/blob/master/README.md)。

##### 安装 Longhorn：

```
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
```

Longhorn 将被安装在命名空间 longhorn-system 中。

##### 创建 PVC 和 pod：

**pvc.yaml**

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-volv-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 2Gi
```

**pod.yaml**

```
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
  namespace: default
spec:
  containers:
  - name: volume-test
    image: nginx:stable-alpine
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: volv
      mountPath: /data
    ports:
    - containerPort: 80
  volumes:
  - name: volv
    persistentVolumeClaim:
      claimName: longhorn-volv-pvc
```

```
kubectl create -f pvc.yaml
kubectl create -f pod.yaml
```

##### 确认 PV 和 PVC 已创建：

```
kubectl get pv
kubectl get pvc
```

## 设置 NFS

如果你的 K3S 集群是 v1.20+，在 nfs provisioner 创建 PersistentVolumeClaim，PersistentVolumeClaim 保持 Pending 状态, 且 nfs provisioner 会报错：

```
I0512 03:01:54.863533       1 controller.go:926] provision "default/v1" class "nfs-provisioner": started
E0512 03:01:54.867892       1 controller.go:943] provision "default/v1" class "nfs-provisioner": unexpected error getting claim reference: selfLink was empty, can't make reference
```

#### 原因

在 k8s 1.20 中，已根据 [release notes](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.20.md) 删除 selfLink 参数。

正如 [github 评论](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/issues/25#issuecomment-742616668) 中所指出的，将 RemoveSelfLink 设置为 false 可以解决这个问题。

#### 解决方法

```
$ curl -sfL https://get.k3s.io | sh -s - --kube-apiserver-arg "feature-gates=RemoveSelfLink=false"
```

或

```
$ cat /etc/systemd/system/k3s.service
...
ExecStart=/usr/local/bin/k3s \
    server \
        '--kube-apiserver-arg' \
        'feature-gates=RemoveSelfLink=false' \
...
```

最后，不要忘记重启 K3s 触发更新。
