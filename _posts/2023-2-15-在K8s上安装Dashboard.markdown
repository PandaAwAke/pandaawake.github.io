---
layout: post
title:  "在K8s上安装Dashboard"
date:   2022-02-15 12:00:00 +0800
categories: Kubernetes Dashboard
---



## 目标

安装 Kubernetes Dashboard。尽可能让读者可以复刻。

官网对Dashboard的安装教程：

[部署和访问 Kubernetes 仪表板（Dashboard）](https://kubernetes.io/zh-cn/docs/tasks/access-application-cluster/web-ui-dashboard/)

但本博客要做一些拓展，以便我们的访问。所以本博客的一部分内容是搬运上述链接的。



## 下载和应用 Dashboard 资源文件

截至今日(2023.2.15)，官方教程建议我们运行下面命令以安装Dashboard：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

但国内无法访问这个地址，如果你想要最新版可以自己想办法下载。

这里给出我的版本（该版本暴露了NodePort 31000端口，还修改了登录过期时间为7天）：

[kube-dashboard.yaml]({{site.url}}/assets/2023-2-15-在K8s上安装Dashboard.assets/kube-dashboard.yaml)



如果你是自己下载的而不是用我的，你可能还需要进行下面两个操作。


### 暴露端口 (可选)

默认是不暴露端口的，如果你需要暴露端口的话找到下面这一段并自行暴露：

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort			# 在这里设置nodePort或其他类型的暴露
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 31000		# 在这里设置端口
  selector:
    k8s-app: kubernetes-dashboard
```



### 修改登录过期时间 (可选)

默认登录过期时间是15min，太短了，每次过期都要重新输token，太麻烦，你可以自己改长一点

```yaml
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.7.0
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            - --token-ttl=604800	# 在这里添加并修改秒数，这里是7天
```



### 应用资源文件

```bash
kubectl apply -f kube-dashboard.yaml
```



## 为 Dashboard 创建访问 Token

仅仅安装Dashboard并暴露端口是不能访问的，进入页面后他还会索要Token。

你可以创建`kube-dashboard-token.yaml`，或下载

[kube-dashboard-token.yaml]({{site.url}}/assets/2023-2-15-在K8s上安装Dashboard.assets/kube-dashboard-token.yaml)

内容如下：

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-secret
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "kubernetes-dashboard"
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard
```

### 应用资源文件

```bash
kubectl apply -f kube-dashboard-token.yaml
```



## 获取 Token 并登录

### 获取 Token

```bash
kubectl describe secret kubernetes-dashboard-secret -n kubernetes-dashboard
```

之后用这个 token 去登录即可。另外注意登录时需要在地址前面加上 `https`。