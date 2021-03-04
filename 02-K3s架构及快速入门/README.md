# K3s架构及快速入门

## 架构介绍

K3s架构：

![](https://tva1.sinaimg.cn/large/008eGmZEly1go0yw51vlgj31iv0u03z7.jpg)

K8s架构：
![](https://d33wubrfki0l68.cloudfront.net/2475489eaf20163ec0f54ddc1d92aa8d4c87c96b/e7c81/images/docs/components-of-kubernetes.svg)

#### 集群架构介绍

中文官网：https://docs.rancher.cn/docs/k3s/architecture/_index

## 快速入门

中文官网：https://docs.rancher.cn/docs/k3s/quick-start/_index

**镜像加速：**

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
