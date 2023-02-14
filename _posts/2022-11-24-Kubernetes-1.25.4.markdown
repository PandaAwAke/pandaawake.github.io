---
layout: post
title:  "Kubernetes 1.25.4 (Virtualbox7.0.4 + Ubuntu22.04 + Containerd) 安装实录与复现教程"
date:   2022-11-24 13:10:00 +0800
categories: Kubernetes Ubuntu 虚拟机
---

这是一篇安装实录：综合了大量的“教程”历尽种种磨难后🤮，终于成功安装 Kubernetes(K8s) 1.25.4，谨以此文防止自己或后人有需之时再受迫害。



## 目标

在 中国互联网环境 内搭建基于 3 台虚拟机 (固定IP) 的 Kubernetes 1.25.4 + Containerd 集群运行环境。



## 环境

这里完整记录环境，以确保读者的复现：

* 主机：Ubuntu 22.04 x64 (Intel Core i7-9750H, 16.0 GB RAM)
* 虚拟机：Virtualbox 7.0.4 + Ubuntu 22.04.1 Live Server amd64 (2-CPU Core, 4.0 GB RAM)
* 软件：Kubernetes 1.25.4 + Containerd
* 网络：墙内中国网 (无法访问部分K8s镜像地址)



## 虚拟机配置

### 安装

在主机 (Ubuntu 22.04 LTS) 下使用 Virtualbox 7.0.4 创建虚拟机，Windows 主机应该没啥区别。

Virtualbox 和扩展包的安装过程应该可以省略，镜像使用 `ubuntu-22.04.1-live-server-amd64.iso`，如果出现问题自己 TroubleShoot 吧（🤪

虚拟机应该是不需要安装增强功能的（虽然我装了）。

### 网络配置 (所有虚拟机)

创建时网络很重要，我换了好几种配置，多少都会出错，最终成功能跑的配置是：

* 网卡1：网络地址转换(NAT)
* 网卡2：仅主机(Host-Only)网络，没有的话去创建一个先，我这边创建的vboxnet是手动配置网卡，且关闭DHCP服务器；我不清楚开启DHCP服务器有无影响。网段如果需要改的话自己更改吧。

我一开始是将两个网卡顺序反过来的，但是后来发现主机ping虚拟机时会产生大量的DUP包，原因不明。

我的三个虚拟机配置如下：

* 虚拟机1：k8s-master 节点，192.168.56.101
* 虚拟机2、3：k8s-worker1、k8s-worker2 节点，192.168.56.102、192.168.56.103

安装好系统后，需要修改 netplan 配置文件，并配置使用。因为设置了两张网卡，虚拟机系统里默认是没完全开启也没配置的。

执行 `sudo vim /etc/netplan/00-installer-config.yaml`，如下修改：

![image-20221124134250904]({{ site.url }}/assets/2022-11-24-Kubernetes-1.25.4.assets/image-20221124134250904.png)

然后执行 `sudo netplan apply` 即可。

最后每个虚拟机要知道彼此的IP，你可以配置hosts文件，例如：

```python
# VM based Kubernetes clusters
192.168.56.101 k8s-master
192.168.56.102 k8s-worker1
192.168.56.103 k8s-worker2
```

也可以自己配置一个DNS服务器，这样更优雅，但我还是比较懒😩，所以算了

### 禁用Swap (所有虚拟机)

```shell
sudo swapoff -a
sudo sed -i '/swap/ s/^\(.*\)$/#\1/g' /etc/fstab	# 注释swap
free -h		# 检查swap是否被关闭，Swap若为0则没问题
```

### 打开内核功能与开启IPv4转发 (所有虚拟机)

```shell
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system	# 重新加载 sysctl
```

### 主机名 (非必须)

使用的主机名和上面hosts文件里的主机名貌似没有必要联系，不需要完全一致，但保持一致更简单。

如果你需要修改每一台虚拟机的主机名，使用类似如下命令即可：

`sudo hostnamectl --static set-hostname k8s-master`

### 换源 (非必须)

这一步也非必须，但在国内的你可以换源以加快速度。

为了方便，给出使用 USTC 源的命令：

`sudo sed -i 's/cn.archive.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list`

### SSH (非必须)

为了方便，你可以给每台虚拟机安装 ssh，甚至按照需求开启 ssh 的 root 登录权限。如果需要开启 root 登录权限以更好的利用 sftp 等工具操作文件或进行其他操作，可以参考 [Ubuntu中开启ssh允许root远程ssh登录的方法](https://cloud.tencent.com/developer/article/1445519)。

### 同步时间 (非必须)

如果时间不同步在 apt update 或 K8s 加入时可能会产生问题，如果发现虚拟机时间严重不同步（可能和主机有时区上的差异，这无伤大雅，重要是虚拟机之间不能有太大差距），需要同步一下时间，这个就自己找找方法吧。



## 安装 Containerd + Kubernetes 1.25.4 (所有虚拟机)

在写这篇博客的时候，Kubernetes 最新版本就是 1.25.4。在这个版本的K8s官方镜像地址貌似从 k8s.gcr.io 变成了 registry.k8s.io，所以在其他地方看见这两个地址需要格外关注，以防被坑。当然，本博客不会在这个地方坑你，因为我都踩过了。。。

### 安装 Containerd

如果你不是 Ubuntu，可能需要参考 [Docker Engine installation overview](https://docs.docker.com/engine/install/)

本部分的剩余部分摘录自 [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

1. 卸载旧版本 docker, containerd 等 (对新装的虚拟机可能不需要)
   `sudo apt-get remove docker docker-engine docker.io containerd runc`

2. 设置 apt 仓库

   ```shell
   sudo apt-get update
   sudo apt-get install \
       ca-certificates \
       curl \
       gnupg \
       lsb-release
   
   sudo mkdir -p /etc/apt/keyrings
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
   
   echo \
     "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
     $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
   ```

3. 更新 apt 包索引，并安装 containerd

   ```shell
   sudo apt-get update
   sudo apt install -y containerd.io
   ```

4. 配置 containerd 用 SystemdCgroup 启动

   ```shell
   containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
   sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
   ```

5. 修改 containerd 内 sandbox_image 的镜像地址为阿里云镜像（官方的访问不到），修改两个地址是保险。修改这个 sandbox_image 的镜像地址是十分必要的，你可以检查一下 /etc/containerd/config.toml，其中的 sandbox_image 是一个名为 pause 的镜像，没有它则 Kubernetes 根本无法拉起集群。

   ```shell
   sudo sed -i 's/registry.k8s.io/registry.aliyuncs.com\/google_containers/g' /etc/containerd/config.toml
   sudo sed -i 's/k8s.gcr.io/registry.aliyuncs.com\/google_containers/g' /etc/containerd/config.toml
   ```

6. 重启并启用 containerd

   ```shell
   sudo systemctl restart containerd
   sudo systemctl enable containerd
   ```



### 安装 Kubernetes

再重复一遍，以防被坑：在写这篇博客的时候，Kubernetes 最新版本就是 1.25.4。如果你想复现，可能需要自己指定版本😬

1. 设置阿里云 K8s apt 仓库

   ```shell
   curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
   sudo apt-add-repository "deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"		# 不建议乱改命令
   
   sudo apt-get update
   # 安装
   ```

2. 安装 kubectl, kubeadm, kubelet

   ```shell
   sudo apt update
   sudo apt install -y kubelet kubeadm kubectl
   或者
   sudo apt install -y kubelet=1.25.4-00 kubeadm=1.25.4-00 kubectl=1.25.4-00
   sudo apt-mark hold kubelet kubeadm kubectl	# 防止它们被自动升级等
   ```



## 利用 kubeadm 拉起与加入 Kubernetes 集群

### 初始化集群 (master节点)

1. 在 K8s 的 master 节点拉起集群
   下面是命令示例：

   ```shell
   sudo kubeadm init \
       --apiserver-advertise-address=192.168.56.101 \
       --image-repository registry.aliyuncs.com/google_containers \
       --pod-network-cidr=10.244.0.0/16
   ```

   - 192.168.56.101 改成你 master 节点的IP地址
   - 可以添加别的配置项，但不建议删除上述任何一个配置项
   - 应该可以直接初始化成功。。。吧😮

   成功后记录一下屏幕上输出的 join 命令，等会要用到（这个应该是有时限的）

2. 让 master 节点在非 root 用户下即可使用 kubectl（这也是集群拉起后kubeadm建议的）

   ```shell
   mkdir -p $HOME/.kube
   cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   chown $(id -u):$(id -g) $HOME/.kube/config
   ```



### 加入集群 (其他节点)

1. 输入加入指令（master节点初始化后输出的指令），加入sudo，例如：

   ```shell
   sudo kubeadm join 192.168.56.101:6443 --token nvg5ar.8gwgupg90rubz92r \
       --discovery-token-ca-cert-hash sha256:8734b4af0f76ffa2bd43259431827f194a32b9b09c1d5d425c335bf83b4972b3
   ```

   我自己测试不需要把主节点的什么 admin.conf 复制到其他节点上，不用试了



### 检查集群状态 (主节点)

```shell
kubectl cluster-info
kubectl get nodes
```

如果加入成功，你会看见每个机子都有一个 node，但所有的机子包括 master 节点都是 NotReady 状态。

需要安装一个 CNI 插件。



## 安装 CNI 插件 (主节点)

所有的教程都让我装 calico，装的方法还都各不相同，反正乱七八糟的最后有的时候是 Ready 了，但我后面再装 Prometheus 之类东西的时候又不能 work，又不知道问题在哪，很崩溃，最后换个 CNI 插件好了

最后我换成了 flannel，安装简单，且后续使用也没出现问题。你可以按照我这个来装，也可以上 calico 官网找找如何安装，这里我就安装 flannel 了。

0. **如果你是重装 K8s，或者之前捣鼓过 CNI 插件，一定要在所有机子上 `sudo rm -rf /etc/cni` 清除干净配置文件，否则你 kubeadm reset 都成功不了**

1. 下载 flannel 资源文件：直接下载或复制 [https://github.com/flannel-io/flannel/blob/master/Documentation/kube-flannel.yml](https://github.com/flannel-io/flannel/blob/master/Documentation/kube-flannel.yml)，写这篇博客的时候 git commit 是 28ed89494c82970f2c1a0763d2bad6bb03a3fed3

2. 修改 flannel 网卡：找到下面这段，在 args 添加 --iface=enp0s8（因为之前虚拟机配置的第二个网卡是Host-Only，且enp0s8就是我虚拟机的第二个网卡，如果你的不一样需要自己找找咯）

   ```yaml
   containers:
   - name: kube-flannel
   #image: flannelcni/flannel:v0.20.1 for ppc64le and mips64le (dockerhub limitations may apply)
     image: docker.io/rancher/mirrored-flannelcni-flannel:v0.20.1
     command:
     - /opt/bin/flanneld
     args:
     - --ip-masq
     - --kube-subnet-mgr
   ```

   改成：

   ```yaml
   containers:
   - name: kube-flannel
   #image: flannelcni/flannel:v0.20.1 for ppc64le and mips64le (dockerhub limitations may apply)
     image: docker.io/rancher/mirrored-flannelcni-flannel:v0.20.1
     command:
     - /opt/bin/flanneld
     args:
     - --ip-masq
     - --kube-subnet-mgr
     - --iface=enp0s8
   ```

3. 应用资源文件：主节点`kubectl apply -f kube-flannel.yml`

之后耐心等待一阵子，你可以用下面这些指令查看 pods 状态：

```shell
kubectl get pods -n kube-system
kubectl get pods -n kube-flannel
```

