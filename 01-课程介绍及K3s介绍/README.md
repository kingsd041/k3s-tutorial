# 课程介绍及 K3s 介绍

## 课程介绍

本教程是 K3s 的系列教程，会按照[K3s 官网](https://docs.rancher.cn/k3s/)的目录顺序来针对 K3s 的每个功能做讲解和操作，以便大家更深入了解 K3s。

当 K3s 新版本发布时，会针对每个版本做详细的介绍。

## K3s 介绍

[K3s - 轻量级 Kubernetes:](https://docs.rancher.cn/docs/k3s/_index)
- CNCF认证的Kubernetes发行版
- 支持 X86_64, ARM64, ARMv7平台
- 单一进程包含Kubernetes master，kubelet，和 containerd


**生命周期:** K3s 会同时支持 3 个 K8s 版本，支持的生命周期与 K8s 相同，参考: [Kubernetes 版本及版本偏差支持策略](https://kubernetes.io/zh/docs/setup/release/version-skew-policy/)

**v1.20.4+k3s1**: `v1.20.4` 为 K8s 版本，`k3s1` 为补丁版本

**更新周期**: 当 K8s 更新新版本后，一般情况下，K3s 在一周内同步更新。

**latest/stable/testing**: https://update.k3s.io/v1-release/channels

## 国内资源

中文官网文档：https://docs.rancher.cn/k3s

国内 K3s 资源：http://mirror.cnrancher.com/

## 周边项目

- [k3os](https://github.com/rancher/k3os): k3OS是一个完全基于Kubernetes管理的轻量级操作系统，能大大简化在低资源环境中的Kubernetes操作，快速安装，秒级启动。
- [k3d](https://github.com/rancher/k3d): k3d创建容器化的k3s集群。这意味着，您可以使用docker在单台计算机上启动多节点k3s集群。
- [autok3s](https://github.com/cnrancher/autok3s): AutoK3s 是用于简化 K3s 集群管理的轻量级工具，您可以使用 AutoK3s 在任何地方运行 K3s，Run K3s Everywhere。
- [octopus](https://github.com/cnrancher/octopus): Octopus是用于Kubernetes和k3s的轻量级云原生设备管理系统，它不需要替换Kubernetes集群的任何基本组件。部署octopus后，集群可以将边缘设备作为自定义k8s资源进行管理。
- [harvester](https://github.com/rancher/harvester): Harvester是基于Kubernetes构建的开源超融合基础架构（HCI）软件。它是vSphere和Nutanix的开源替代方案。

## 遇到问题怎么办？

1. k3s github issue: https://github.com/k3s-io/k3s/issues
2. 在此repo中创建issue
3. 中文论坛: https://forums.cnrancher.com/
4. Rancher 社区:
![](https://tva1.sinaimg.cn/large/008eGmZEly1go02ru4c4yj30r60e4wi5.jpg)