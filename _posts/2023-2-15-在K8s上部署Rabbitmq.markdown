---
layout: post
title:  "在K8s上部署Rabbitmq"
date:   2022-02-15 12:00:00 +0800
categories: Kubernetes Rabbitmq
---



## 目标

在 K8s 上部署 Rabbitmq 集群。尽可能让读者可以复刻。

rabbitmq 官网的部署教程：

[RabbitMQ Cluster Operator for Kubernetes — RabbitMQ](https://www.rabbitmq.com/kubernetes/operator/operator-overview.html)



## 部署 RabbitMQ Cluster Operator

截至今日(2023.2.15)，官方教程建议我们运行下面命令：

```bash
kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml"
# namespace/rabbitmq-system created
# customresourcedefinition.apiextensions.k8s.io/rabbitmqclusters.rabbitmq.com created
# serviceaccount/rabbitmq-cluster-operator created
# role.rbac.authorization.k8s.io/rabbitmq-cluster-leader-election-role created
# clusterrole.rbac.authorization.k8s.io/rabbitmq-cluster-operator-role created
# rolebinding.rbac.authorization.k8s.io/rabbitmq-cluster-leader-election-rolebinding created
# clusterrolebinding.rbac.authorization.k8s.io/rabbitmq-cluster-operator-rolebinding created
# deployment.apps/rabbitmq-cluster-operator created/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

它创建了新的 Namespace (rabbitmq-system)，一个 deployment 被创建，同时定义了资源 rabbitmqclusters.rabbitmq.com，以便我们创建集群使用。



## 设置并创建PV Provisioner (非生产环境)

现在创建 Rabbitmq 集群是跑不起来的，因为没有 PV(Physical Volume)。可以安装 [Local Path Provisioner](https://github.com/rancher/local-path-provisioner) 在**非生产环境**下解决这个问题。

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl annotate storageclass local-path storageclass.kubernetes.io/is-default-class=true
```

老样子，这个github raw地址访问不到，你可能需要自己想办法去弄一份，也可以下载[local-path-storage.yaml]({{site.url}}/assets/2023-2-15-在K8s上部署Rabbitmq.assets/local-path-storage.yaml)代替



## 设置并创建 RabbitMQ 集群

定义参考：[Using RabbitMQ Cluster Kubernetes Operator — RabbitMQ](https://www.rabbitmq.com/kubernetes/operator/using-operator.html)

自己创建一个资源文件以定义集群属性（可以限制资源），例如：

```bash
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
    name: 你的rabbitmq集群名
spec:
  replicas: 容器副本数
  resources:	# 单个容器的资源限制，默认给了1CPU和2GB内存，下面是自定义的例子
    requests:
      cpu: 800m
      memory: 1000Mi
    limits:
      cpu: 800m
      memory: 1000Mi
  service:
    type: NodePort
```

这里部署的 RabbitMQ Server 没指定 Namespace 的话是在 default namespace 的。

我没有仔细研究，貌似这里的 NodePort 是不能指定端口的，后面你可以自己用`kubectl get svc`查一下15672对应暴露的端口。



## 访问并登录 RabbitMQ 面板页面

```bash
username="$(kubectl get secret hello-world-default-user -o jsonpath='{.data.username}' | base64 --decode)"
password="$(kubectl get secret hello-world-default-user -o jsonpath='{.data.password}' | base64 --decode)"


echo "username: $username"
echo "password: $password"
```

然后用这个用户名和密码登录即可。