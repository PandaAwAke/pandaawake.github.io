---
layout: post
title:  "在 Kubernetes 1.25.4 集群中安装 Prometheus 2.40.2 安装实录与复现教程"
date:   2022-11-24 13:10:00 +0800
categories: Kubernetes Ubuntu 虚拟机
---



## 目标

在 "中国互联网环境 内搭建的基于 3 台虚拟机 (固定IP) 的 Kubernetes 1.25.4 + Containerd 的集群运行环境" 中安装 Prometheus 2.40.2 + Grafana。

K8s 集群的安装是基于 [Kubernetes 1.25.4 (Virtualbox7.0.4 + Ubuntu22.04 + Containerd) 安装实录与复现教程]({% post_url 2022-11-24-Kubernetes-1.25.4 %}) 进行的。



## 环境

这里完整记录环境，以确保读者的复现：

* 主机：Ubuntu 22.04 x64 (Intel Core i7-9750H, 16.0 GB RAM)
* 虚拟机：Virtualbox 7.0.4 + Ubuntu 22.04.1 Live Server amd64 (2-CPU Core, 4.0 GB RAM)
* 软件：Kubernetes 1.25.4 + Containerd
* 网络：墙内中国网 (无法访问部分K8s镜像地址)

要安装的 Kubernetes 资源：

* https://github.com/prometheus-operator/kube-prometheus

在笔者实装时，kube-prometheus 的 git commit 为 35f69e8b03bdfed476769e9e0992bd750a393387。



## 注明

所有操作均在主节点完成，其他节点不需要进行任何操作。



## 克隆仓库

这一步很简单，克隆的是 kube-prometheus 仓库。

```shell
git clone https://github.com/prometheus-operator/kube-prometheus
cd kube-prometheus
```



## 修改镜像源

在应用 Prometheus 的资源配置文件之前，需要进行修改，这是因为 Prometheus 的配置文件内有两个主要的镜像仓库地址，但国内都无法访问。

先进入 manifests 文件夹，这里为需要部署的所有 K8s 资源文件：

```shell
cd manifests
```

* quay.io：国内用的较多的镜像源替代有

  * quay.mirrors.ustc.edu.cn
  * quay-mirror.qiniu.com

  本博客选择 ustc 源。

* registry.k8s.io (或k8s.gcr.io)

  * 和之前写的 K8s 集群安装博客一样，新版的 K8s 貌似将镜像仓库从 k8s.gcr.io 改为了 registry.k8s.io
  * manifests 里的资源文件有一些使用了该地址的镜像，但无奈的是没有镜像源
    * 在部署 K8s 集群的时候，你也许有印象我们把 registry.k8s.io 或 k8s.gcr.io 改为了 registry.aliyuncs.com/google_containers，对吧？但那是 google_containers，Prometheus 确实没有镜像，至少笔者没有找到
  * 本博客采取的解决方案是：去 DockerHub 上查找相同镜像
    * 本博客 clone 下来的 manifests 资源文件中，出现的两个 registry.k8s.io 镜像为：
      * (kubeStateMetrics-deployment.yaml) registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.6.0
      * (prometheusAdapter-deployment.yaml) registry.k8s.io/prometheus-adapter/prometheus-adapter:v0.10.0
    * 去 DockerHub 搜索这两个镜像，搜到的替代地址为：
      * bitnami/kube-state-metrics:2.6.0
      * v5cn/prometheus-adapter:v0.10.0
    * 修改上面的两个文件 kubeStateMetrics-deployment.yaml 和 prometheusAdapter-deployment.yaml，分别修改地址为：
      * docker.io/bitnami/kube-state-metrics:2.6.0
      * docker.io/v5cn/prometheus-adapter:v0.10.0

最后，即执行：

```shell
# 替换 quay.io
sed -i "s#quay.io#quay.mirrors.ustc.edu.cn#g" *.yaml

# 替换 registry.k8s.io
sed -i "s#registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.6.0#docker.io/bitnami/kube-state-metrics:2.6.0#g" kubeStateMetrics-deployment.yaml
sed -i "s#registry.k8s.io/prometheus-adapter/prometheus-adapter:v0.10.0#docker.io/v5cn/prometheus-adapter:v0.10.0#g" prometheusAdapter-deployment.yaml
```



## 修改配置文件以暴露服务 (非必须)

默认的 manifests 资源文件没有暴露 pods 服务，你可以通过修改类型为 NodePort 以暴露 grafana、prometheus-service 和 alertmanager-service。

```shell
# 暴露 grafana
sed -i  "/ports:/i\  type: NodePort" grafana-service.yaml
sed -i  "/targetPort: http/i\    nodePort: 31100" grafana-service.yaml

# 暴露 prometheus-service
sed -i  "/ports:/i\  type: NodePort" prometheus-service.yaml
sed -i  "/targetPort: web/i\    nodePort: 31200" prometheus-service.yaml
sed -i  "/targetPort: reloader-web/i\    nodePort: 31300" prometheus-service.yaml

# 暴露 alertmanager-service
sed -i  "/ports:/i\  type: NodePort" alertmanager-service.yaml
sed -i  "/targetPort: web/i\    nodePort: 31400" alertmanager-service.yaml
sed -i  "/targetPort: reloader-web/i\    nodePort: 31500" alertmanager-service.yaml
```

进行上述操作后，集群端口 31100、31200、31300、31400、31500 将暴露，你可以自己修改端口。



## 应用资源文件

```shell
# 假设你在 manifests 文件夹下
kubectl apply --server-side -f ./setup
kubectl wait \
	--for condition=Established \
	--all CustomResourceDefinition \
	--namespace=monitoring
kubectl apply -f .
```



## 检查状态

```shell
kubectl get pods -n monitoring
```



## 参考资料

https://cloud.tencent.com/developer/article/1986296

