# K3s架构及快速入门

## K3s架构

#### 如何运行的

![](https://tva1.sinaimg.cn/large/008eGmZEly1go0yw51vlgj31iv0u03z7.jpg)

#### 架构介绍

中文官网：https://docs.rancher.cn/docs/k3s/architecture/_index

## 快速入门

中文官网：https://docs.rancher.cn/docs/k3s/quick-start/_index

**镜像加速：**

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://fogjl973.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload
systemctl restart docker
```
