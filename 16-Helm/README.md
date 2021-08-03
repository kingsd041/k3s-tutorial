# Helm

Helm 是 Kubernetes 的包管理工具。Helm Chart 为 Kubernetes YAML 清单文件提供了模板化语法，可以通过 Helm 安装对应的chart。更多请参考：[Helm 快速入门](https://helm.sh/docs/intro/quickstart/)

K3s 不需要任何特殊的配置就可以使用 Helm 命令行工具。只要确保你已经按照[集群访问](http://docs.rancher.cn/docs/k3s/cluster-access/_index/)一节正确设置了你的 kubeconfig。 K3s 通过 rancher/helm-release CRD 使传统的 Kubernetes 资源清单和 Helm Charts 部署更加容易。

## 自动部署 Helm charts

在`/var/lib/rancher/k3s/server/manifests`中找到的任何 Kubernetes 清单将以类似`kubectl apply`的方式自动部署到 K3s。以这种方式部署的 manifests 是作为 AddOn 自定义资源来管理的，可以通过运行`kubectl get addon -A`来查看。你会发现打包组件的 AddOns，如 CoreDNS、Local-Storage、Traefik 等。AddOns 是由部署控制器自动创建的，并根据它们在 manifests 目录下的文件名命名。

也可以将 Helm Chart 作为 AddOns 部署。K3s 包括一个[Helm Controller](https://github.com/rancher/helm-controller/)，它使用 HelmChart Custom Resource Definition(CRD)管理 Helm Chart。

## 使用 Helm CRD

[HelmChart CRD](https://github.com/rancher/helm-controller#helm-controller)捕获了大多数你通常会传递给`helm`命令行工具的选项。下面是一个例子，说明如何从默认的 Chart 资源库中部署 Grafana，覆盖一些默认的 Chart 值。请注意，HelmChart 资源本身在 `kube-system` 命名空间，但 Chart 资源将被部署到 `monitoring` 命名空间。

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: grafana
  namespace: kube-system
spec:
  chart: stable/grafana
  targetNamespace: monitoring
  set:
    adminPassword: "NotVerySafePassword"
  valuesContent: |-
    image:
      tag: master
    env:
      GF_EXPLORE_ENABLED: true
    adminUser: admin
    sidecar:
      datasources:
        enabled: true
```

### HelmChart 字段定义

| 字段                 | 默认值  | 描述                                                                        | Helm Argument / Flag Equivalent |
| :------------------- | :------ | :-------------------------------------------------------------------------- | :------------------------------ |
| name                 | N/A     | Helm Chart 名称                                                             | NAME                            |
| spec.chart           | N/A     | 仓库中的 Helm Chart 名称，或完整的 HTTPS URL（.tgz）。                      | CHART                           |
| spec.targetNamespace | default | Helm Chart 目标命名空间                                                     | `--namespace`                   |
| spec.version         | N/A     | Helm Chart 版本(从版本库安装时使用的版本号)                                 | `--version`                     |
| spec.repo            | N/A     | Helm Chart 版本库 URL 地址                                                  | `--repo`                        |
| spec.helmVersion     | v3      | Helm 的版本号，可选值为 `v2` 和`v3`，默认值为 `v3`                          | N/A                             |
| spec.set             | N/A     | 覆盖简单的默认 Chart 值。这些值优先于通过 valuesContent 设置的选项。        | `--set` / `--set-string`        |
| spec.jobImage        |         | 指定安装 helm chart 时要使用的镜像。如：rancher/klipper-helm:v0.3.0。       |                                 |
| spec.valuesContent   | N/A     | 通过 YAML 文件内容覆盖复杂的默认 Chart 值。                                 | `--values`                      |
| spec.chartContent    | N/A     | Base64 编码的 Chart 存档.tgz - 覆盖 spec.chart。                            | CHART                           |

放在`/var/lib/rancher/k3s/server/static/`中的内容可以通过 Kubernetes APIServer 从集群内匿名访问。这个 URL 可以使用`spec.chart`字段中的特殊变量`%{KUBERNETES_API}%`进行模板化。例如，打包后的 Traefik 组件从`https://%{KUBERNETES_API}%/static/charts/traefik-1.81.0.tgz`加载其 Chart。

## 使用 HelmChartConfig 自定义打包的组件

为了允许覆盖作为 HelmCharts（如 Traefik或其他通过helm crd 部署的应用）部署的打包组件的值，从 v1.19.0+k3s1 开始的 K3s 版本支持通过 HelmChartConfig CRD 部署。HelmChartConfig 资源必须与其对应的 HelmChart 的名称和命名空间相匹配，并支持提供额外的 "valuesContent"，它作为一个额外的值文件传递给`helm`命令。

> **注意：** HelmChart 的`spec.set`值覆盖了 HelmChart 和 HelmChartConfig 的`spec.valuesContent`设置。

```yaml
apiVersion: helm.cattle.io/v1
kind: HelmChartConfig
metadata:
  name: grafana
  namespace: kube-system
spec:
  valuesContent: |-
    service:
      type: NodePort
```

如果要自定义打包后的 Traefik 入口配置，你可以创建一个名为`/var/lib/rancher/k3s/server/manifests/traefik-config.yaml`的文件，并将其填充为以下内容。
