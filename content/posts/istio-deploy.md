---
keywords:
- 米开朗基杨
- istio
- istio cni
- critical pod
title: "Istio 1.4 部署指南"
subtitle: "使用 Istio CNI Plugin 来转发流量"
description: 本文将会告诉你如何使用 istioctl 部署 istio 1.4，并开启 istio CNI 插件。
date: 2019-12-15T00:14:06+08:00
draft: false
toc: true
categories: "service mesh"
tags: ["istio", "service mesh"]
img: "https://hugo-picture.oss-cn-beijing.aliyuncs.com/images/20191215002051.png"
bigimg: [{src: "https://hugo-picture.oss-cn-beijing.aliyuncs.com/blog/2019-04-27-080627.jpg"}]
---

Istio 一直处于快速迭代更新的过程中，它的部署方法也在不断更新，之前我在 1.0 版本中介绍的安装方法，对于最新的 1.4 版本已经不适用了。以后主流的部署方式都是用 istioctl 进行部署，helm 可以渐渐靠边站了~~

在部署 Istio 之前，首先需要确保 Kubernetes 集群（kubernetes 版本建议在 `1.13` 以上）已部署并配置好本地的 kubectl 客户端。

## <span id="inline-toc">1.</span> Kubernetes 环境准备

为了快速准备 kubernetes 环境，我们可以使用 sealos 来部署，步骤如下：

### 前提条件

+ 下载[kubernetes 离线安装包](http://store.lameleg.com/)
+ 下载[最新版本sealos](https://github.com/fanux/sealos/releases)
+ 务必同步服务器时间
+ 主机名不可重复

### 安装 kubernetes 集群

```bash
$ sealos init --master 192.168.0.2 \
    --node 192.168.0.3 \
    --node 192.168.0.4 \
    --node 192.168.0.5 \
    --user root \
    --passwd your-server-password \
    --version v1.16.3 \
    --pkg-url /root/kube1.16.3.tar.gz 
```

检查安装是否正常：

```bash
$ kubectl get node

NAME       STATUS   ROLES    AGE   VERSION
sealos01   Ready    master   18h   v1.16.3
sealos02   Ready    <none>   18h   v1.16.3
sealos03   Ready    <none>   18h   v1.16.3
sealos04   Ready    <none>   18h   v1.16.3
```

## <span id="inline-toc">2.</span> 下载 Istio 部署文件

你可以从 GitHub 的 [release](https://github.com/istio/istio/releases/tag/1.4.2) 页面下载 istio，或者直接通过下面的命令下载：

```bash
$ curl -L https://istio.io/downloadIstio | sh -
```

下载完成后会得到一个 `istio-1.4.2` 目录，里面包含了：

+ `install/kubernetes` : 针对 Kubernetes 平台的安装文件
+ `samples` : 示例应用
+ `bin` : istioctl 二进制文件，可以用来手动注入 sidecar proxy

进入 `istio-1.4.2` 目录。

```bash
$ cd istio-1.4.2

$ tree -L 1 ./
./
├── bin
├── demo.yaml
├── install
├── LICENSE
├── manifest.yaml
├── README.md
├── samples
└── tools

4 directories, 4 files
```

将 istioctl 拷贝到 `/usr/local/bin/` 中：

```bash
$ cp bin/istioctl /usr/local/bin/
```

### 开启 istioctl 的自动补全功能

#### bash

将 `tools` 目录中的 `istioctl.bash` 拷贝到 $HOME 目录中：

```bash
$ cp tools/istioctl.bash ~/
```

在 `~/.bashrc` 中添加一行：

```bash
source ~/istioctl.bash
```

应用生效：

```bash
$ source ~/.bashrc
```

#### zsh

将 `tools` 目录中的 `_istioctl` 拷贝到 $HOME 目录中：

```bash
$ cp tools/_istioctl ~/
```

在 `~/.zshrc` 中添加一行：

```bash
source ~/_istioctl
```

应用生效：

```bash
$ source ~/.zshrc
```

## <span id="inline-toc">3.</span> 部署 Istio

istioctl 提供了多种安装配置文件，可以通过下面的命令查看：

```bash
$ istioctl profile list

Istio configuration profiles:
    minimal
    remote
    sds
    default
    demo
```

它们之间的差异如下：

|   | default | demo | minimal | sds | remote |
| :---: | :--- |:--- | :--- | :--- | :--- |
| **核心组件** |  |  |  |  |  |
| istio-citadel | **X** | **X** |  | **X** | **X** |
| istio-egressgateway |  | **X** |  |  |  |
| istio-galley | **X** | **X** |  | **X** |  |
| istio-ingressgateway | **X** | **X** |  | **X** |  |
| istio-nodeagent |  |  |  | **X** |  |
| istio-pilot | **X** | **X** | **X** | **X** |  |
| istio-policy | **X** | **X** |  | **X** |  |
| istio-sidecar-injector | **X** | **X** |  | **X** | **X** |
| istio-telemetry | **X** | **X** |  | **X** |  |
| **附加组件** |  |  |  |  |  |
| Grafana |  | **X** |  |  |  |
| istio-tracing |  | **X** |  |  |  |
| kiali |  | **X** |  |  |  |
| prometheus | **X** | **X** |  | **X** |  |

其中标记 **X** 表示该安装该组件。

如果只是想快速试用并体验完整的功能，可以直接使用配置文件 `demo` 来部署。

在正式部署之前，需要先说明两点：

### Istio CNI Plugin

当前实现将用户 pod 流量转发到 proxy 的默认方式是使用 privileged 权限的 `istio-init` 这个 init container 来做的（运行脚本写入 iptables），需要用到 `NET_ADMIN` capabilities。对 linux capabilities 不了解的同学可以参考我的 [Linux capabilities 系列](https://fuckcloudnative.io/posts/linux-capabilities-why-they-exist-and-how-they-work/)。

Istio CNI 插件的主要设计目标是消除这个 privileged 权限的 init container，换成利用 Kubernetes CNI 机制来实现相同功能的替代方案。具体的原理就是在 Kubernetes CNI 插件链末尾加上 Istio 的处理逻辑，在创建和销毁 pod 的这些 hook 点来针对 istio 的 pod 做网络配置：写入 iptables，让该 pod 所在的 network namespace 的网络流量转发到 proxy 进程。

详细内容请参考[官方文档](https://istio.io/docs/setup/additional-setup/cni/)。

使用 Istio CNI 插件来创建 sidecar iptables 规则肯定是未来的主流方式，不如我们现在就尝试使用这种方法。

### Kubernetes 关键插件（Critical Add-On Pods）

众所周知，Kubernetes 的核心组件都运行在 master 节点上，然而还有一些附加组件对整个集群来说也很关键，例如 DNS 和 metrics-server，这些被称为**关键插件**。一旦关键插件无法正常工作，整个集群就有可能会无法正常工作，所以 Kubernetes 通过优先级（PriorityClass）来保证关键插件的正常调度和运行。要想让某个应用变成 Kubernetes 的**关键插件**，只需要其 `priorityClassName` 设为 `system-cluster-critical` 或 `system-node-critical`，其中 `system-node-critical` 优先级最高。

> 注意：关键插件只能运行在 `kube-system` namespace 中！

详细内容可以参考[官方文档](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)。

接下来正式安装 istio，命令如下：

```bash
$ istioctl manifest apply --set profile=demo \
   --set cni.enabled=true --set cni.components.cni.namespace=kube-system \
   --set values.gateways.istio-ingressgateway.type=ClusterIP
```

istioctl 支持两种 API：

+ [IstioControlPlane API](https://istio.io/docs/reference/config/istio.operator.v1alpha12.pb/)
+ [Helm API](https://istio.io/docs/reference/config/installation-options/)

在上述安装命令中，cni 参数使用的是 IstioControlPlane API，而 `values.*` 使用的是 Helm API。

部署完成后，查看各组件状态：

```bash
$ kubectl -n istio-system get pod

NAME                                      READY   STATUS    RESTARTS   AGE
grafana-6b65874977-8psph                  1/1     Running   0          36s
istio-citadel-86dcf4c6b-nklp5             1/1     Running   0          37s
istio-egressgateway-68f754ccdd-m87m8      0/1     Running   0          37s
istio-galley-5fc6d6c45b-znwl9             1/1     Running   0          38s
istio-ingressgateway-6d759478d8-g5zz2     0/1     Running   0          37s
istio-pilot-5c4995d687-vf9c6              0/1     Running   0          37s
istio-policy-57b99968f-ssq28              1/1     Running   1          37s
istio-sidecar-injector-746f7c7bbb-qwc8l   1/1     Running   0          37s
istio-telemetry-854d8556d5-6znwb          1/1     Running   1          36s
istio-tracing-c66d67cd9-gjnkl             1/1     Running   0          38s
kiali-8559969566-jrdpn                    1/1     Running   0          36s
prometheus-66c5887c86-vtbwb               1/1     Running   0          39s
```

```bash
$ kubectl -n kube-system get pod -l k8s-app=istio-cni-node

NAME                   READY   STATUS    RESTARTS   AGE
istio-cni-node-k8zfb   1/1     Running   0          10m
istio-cni-node-kpwpc   1/1     Running   0          10m
istio-cni-node-nvblg   1/1     Running   0          10m
istio-cni-node-vk6jd   1/1     Running   0          10m
```

可以看到 cni 插件已经安装成功，查看配置是否已经追加到 CNI 插件链的末尾：

```bash
$ cat /etc/cni/net.d/10-calico.conflist

{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
  ...
    {
      "type": "istio-cni",
      "log_level": "info",
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/ZZZ-istio-cni-kubeconfig",
        "cni_bin_dir": "/opt/cni/bin",
        "exclude_namespaces": [
          "istio-system"
        ]
      }
    }
  ]
}
```

默认情况下除了 `istio-system` namespace 之外，istio cni 插件会监视其他所有 namespace 中的 Pod，然而这并不能满足我们的需求，更严谨的做法是让 istio CNI 插件至少忽略 `kube-system`、`istio-system` 这两个 namespace，怎么做呢？

也很简单，还记得之前提到的 IstioControlPlane API 吗？可以直接通过它来覆盖之前的配置，只需要创建一个 `IstioControlPlane` CRD 就可以了。例如：

```yaml
$ cat cni.yaml

apiVersion: install.istio.io/v1alpha2
kind: IstioControlPlane
spec:
  cni:
    enabled: true
    components:
      namespace: kube-system
  values:
    cni:
      excludeNamespaces:
       - istio-system
       - kube-system
       - monitoring
  unvalidatedValues:
    cni:
      logLevel: info
```

```bash
$ istioctl manifest apply -f cni.yaml
```

删除所有的 istio-cni-node Pod：

```bash
$ kubectl -n kube-system delete pod -l k8s-app=istio-cni-node
```

再次查看 CNI 插件链的配置：

```bash
$ cat /etc/cni/net.d/10-calico.conflist

{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
  ...
    {
      "type": "istio-cni",
      "log_level": "info",
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/ZZZ-istio-cni-kubeconfig",
        "cni_bin_dir": "/opt/cni/bin",
        "exclude_namespaces": [
          "istio-system",
          "kube-system",
          "monitoring"
        ]
      }
    }
  ]
}
```

## <span id="inline-toc">4.</span> 暴露 Dashboard

这个没什么好说的，通过 Ingress Controller 暴露就好了，可以参考我以前写的 [Istio 1.0 部署](https://fuckcloudnative.io/posts/istio-1.0-deploy/)。如果使用 Contour 的可以参考我的另一篇文章：[Contour 学习笔记（一）：使用 Contour 接管 Kubernetes 的南北流量](https://fuckcloudnative.io/posts/use-envoy-as-a-kubernetes-ingress/)。

这里我再介绍一种新的方式，`istioctl` 提供了一个子命令来从本地打开各种 Dashboard：

```bash
$ istioctl dashboard --help

Access to Istio web UIs

Usage:
  istioctl dashboard [flags]
  istioctl dashboard [command]

Aliases:
  dashboard, dash, d

Available Commands:
  controlz    Open ControlZ web UI
  envoy       Open Envoy admin web UI
  grafana     Open Grafana web UI
  jaeger      Open Jaeger web UI
  kiali       Open Kiali web UI
  prometheus  Open Prometheus web UI
  zipkin      Open Zipkin web UI
```

例如，要想在本地打开 Grafana 页面，只需执行下面的命令：

```bash
$ istioctl dashboard grafana
http://localhost:36813
```

咋一看可能觉得这个功能很鸡肋，我的集群又不是部署在本地，而且这个命令又不能指定监听的 IP，在本地用浏览器根本打不开呀！其实不然，你可以在本地安装 kubectl 和 istioctl 二进制文件，然后通过 kubeconfig 连接到集群，最后再在本地执行上面的命令，就可以打开页面啦，开发人员用来测试是不是很方便？Windows 用户当我没说。。。

## <span id="inline-toc">5.</span> 暴露 Gateway

为了暴露 Ingress Gateway，我们可以使用 `HostNetwork` 模式运行，但你会发现无法启动 ingressgateway 的 Pod，因为如果 Pod 设置了 `HostNetwork=true`，则 dnsPolicy 就会从 `ClusterFirst` 被强制转换成 `Default`。而 Ingress Gateway 启动过程中需要通过 DNS 域名连接 `pilot` 等其他组件，所以无法启动。

我们可以通过强制将 `dnsPolicy` 的值设置为 `ClusterFirstWithHostNet` 来解决这个问题，详情参考：[Kubernetes DNS 高阶指南](https://fuckcloudnative.io/posts/kubernetes-dns/)。

修改后的 ingressgateway deployment 配置文件如下：

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: istio-ingressgateway
  namespace: istio-system
  ...
spec:
  ...
  template:
    metadata:
    ...
    spec:
      affinity:
        nodeAffinity:
          ...
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - 192.168.0.4   # 假设你想调度到这台主机上
      ...
      dnsPolicy: ClusterFirstWithHostNet
      hostNetwork: true
      restartPolicy: Always
      ...
```

接下来我们就可以在浏览器中通过 Gateway 的 URL 来访问服务网格中的服务了。后面我会开启一系列实验教程，本文的所有步骤都是为后面做准备，如果想跟着我做后面的实验，请务必做好本文所述的准备工作。
