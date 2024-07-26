---
title: kubernetes部署记录
tags:
  - k8s
  - kubeadm
categories:
  - - k8s
    - kubeadm
date: 2024-07-26 16:03:06
---


> 本文记录笔者利用kubeadm部署k8s集群中的一些小坑，以作备忘
>
> kubernetes versions: v1.30.0
>
> OS: almalinux 9.4, centos 7
> 
> master: alma9-minikube
> worknode: centos7-minikube
>
> 此外，笔者实验环境设置了各种变量、别名和sudoers继承的环境变量。建议以下操作使用root而不是普通用户

笔者按照kubernetes官网的文档使用kubeadm进行集群部署。使用almalinux 9.4 作为master节点， centos 7 作为工作节点部署一个简单的kubernetes集群

节点信息:
```bash
[baka@routerAlmaLinux ~]$ ansi kube1 'fastfetch --logo none'
centos7-minikube | CHANGED | rc=0 >>
baka@centos7-minikube
---------------------
OS: CentOS Linux 7 x86_64
Host: VirtualBox (1.2)
Kernel: 3.10.0-1160.119.1.el7.x86_64
Uptime: 36 mins
Packages: 448 (rpm)
Shell: bash 4.2.46
Cursor: Adwaita
Terminal: python
CPU: AMD Ryzen 9 3900X (2)
GPU: VMware SVGA II Adapter
Memory: 374.31 MiB / 3.70 GiB (9%)
Disk (/): 3.79 GiB / 21.49 GiB (17%) - xfs
Battery: 100% [Full]
Locale: en_US.UTF-8
alma9-minikube | CHANGED | rc=0 >>
baka@alma9-minikube
-------------------
OS: AlmaLinux 9.4 x86_64
Host: VirtualBox (1.2)
Kernel: Linux 5.14.0-427.24.1.el9_4.x86_64
Uptime: 36 mins
Shell: python3
Display (Virtual-1): 1280x800
Terminal: /dev/pts/2
CPU: AMD Ryzen 9 3900X (2)
GPU: VMware SVGA II Adapter
Memory: 1.07 GiB / 3.57 GiB (30%)
Swap: Disabled
Disk (/): 5.68 GiB / 16.93 GiB (34%) - xfs
Local IP (enp0s3): 192.168.56.90/24
Battery: 100% [AC Connected]
Locale: C.UTF-8
```

# 前言
在部署的集群的过程中，使用kubeadm并参照官方文档一步一步来很快就能起一个集群。但是过程中有一些小细节文档没有提。例如cri换成`非containerd`时，cni为`flannel`时，`kubeadm init/join`需要添加的额外的参数；另外，对于`cri-dockerd`这个容器运行时，怎么把cgroup驱动换成`systemd`

本文将记录:
 - 集群master节点使用cri为`containerd`时开启内核模块`bf_netfilter`并配置相关内核参数
 - 集群cni使用`flannel`时，如何在使用kubeadm初始化集群前/后配置集群的`podCIDR`
 - 工作节点的cri为`cri-dockerd`时，如何将cri的gcroup驱动换成systemd
 - 工作节点的cri为`cri-dockerd`时，且集群cni为`flannel`时如何使用kubeadm加入集群
 
# 实践

## 开启内核模块`bf_netfilter`并配置相关参数

假设当前集群并未初始化，并且master节点已经完成 cri `containerd`的安装

不确定不开启会影响什么，可能与cni有关
> https://unix.stackexchange.com/questions/720105/what-is-the-net-bridge-bridge-nf-call-iptables-kernel-parameter

根据`nerdctl info`的提示:
```bash
[root@cluster2-alma9-master repos]# nerdctl info
Client:
 Namespace:     default
 Debug Mode:    false

Server:
 Server Version: v1.7.19
 Storage Driver: overlayfs
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Log: fluentd journald json-file syslog
  Storage: native overlayfs
 Security Options:
  seccomp
   Profile: builtin
  cgroupns
 Kernel Version: 5.14.0-427.26.1.el9_4.x86_64
 Operating System: AlmaLinux 9.4 (Seafoam Ocelot)
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 3.574GiB
 Name: cluster2-alma9-master
 ID: a02743a9-c219-4509-b660-ca0c916d0f93

WARNING: bridge-nf-call-iptables is disabled
WARNING: bridge-nf-call-ip6tables is disableds
```

### 开启内核模块并配置相关参数

开启内核模块`bf_netfilter`:
```bash
sudo modprobe bf_netfilter
```

开启cni需要的内核参数:
```bash
sudo cat >> /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl -p /etc/sysctl.d
```



## 集群cni为`flannel`时初始化前/后配置集群podCIDR

如果不配置poCIDR或者修改flannel插件的yaml配置，直接安装flannel，flannel的pod将会CrashLoopBackoff

假设这时候按照官方文档部署master节点，使用containerd作为cri，使用flannel作为集群cni，并且集群主节点先决条件准备完毕

### 初始化前

声明式配置:
```bash
# 配置声明式文件
# 在版本 1.22 及更高版本中，如果用户没有在 KubeletConfiguration 中
# 设置 cgroupDriver 字段， kubeadm 会将它设置为默认值 systemd
# 改自:https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/#configuring-the-kubelet-cgroup-driver
cat > k8s-master-init.yaml << EOF
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
kubernetesVersion: v1.30.0
networking:
  podSubnet: 10.244.0.0/16
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
EOF

# 使用上述声明式文件初始化集群
kubeadm --config ./k8s-master-init.yaml

# 安装flannel插件
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

> 声明式api参考:
>
> [kubeadm Configuration (v1beta3)](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta3)
> 
> 本文撰写于:2024-07-25

命令式配置:
```bash
kubeadm init --pod-network-cidr 10.244.0.0/16

# 安装flannel插件
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

检查配置是否生效:
```bash
kubectl cluster-info dump | grep podCIDR
```

### 初始化后

初始化后需要配置podCIDR也不是不行，但是需要的步骤略微麻烦

> 本节参考:
>
> [\[stackoverflow\] Is there a way to assign pod-network-cidr in kubeadm after initialization?](https://stackoverflow.com/questions/60940447/is-there-a-way-to-assign-pod-network-cidr-in-kubeadm-after-initialization)
>
>[kubernetes-api reference doc v1.30](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#node-v1-core)
> 
> [Deploying Flannel with kubectl](https://github.com/flannel-io/flannel#deploying-flannel-manually)

首先需要进入master节点编辑node(如果有worknodes，那么worknodes也得改)的配置文件:
```bash
[baka@cluster2-alma9-master ~]$ kubectl get nodes
NAME                    STATUS     ROLES           AGE   VERSION
cluster2-alma9-master   NotReady   control-plane   20h   v1.30.3
[baka@cluster2-alma9-master ~]$ kubectl edit node cluster2-alma9-master
```

> 之后就是使用默认编辑器编辑nodes声明式配置的界面，找到spec.podCIDR
> 
> 其他字段参考[kubernetes-api reference doc v1.30](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#node-v1-core)

```yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
......
spec: 
  podCIDR: 10.244.0.0/24
......
```

在各个节点上删除受影响的接口:
```bash
sudo ip link del cni0; sudo ip link del flannel.1
```

重新起插件和coredns的pod:
```bash
kubectl delete pod --selector=app=flannel -n=kube-flannel
kubectl delete pod --selector=k8s-app=kube-dns -n kube-system
```

检查配置是否生效:
```bash
kubectl get pods -A
kubectl cluster-info dump | grep podCIDR
```

## 工作节点的cri为cri-dockerd时，将cri的gcroup驱动换成systemd

在运行远古操作系统时，dockerd的cgroup驱动默认为`cgroupfs`。既然init是systemd,那么按照kubernetes的文档，自然换成systemd会好点

> 本节参考:
> 
> https://kubernetes.io/zh-cn/docs/concepts/architecture/cgroups/
> 
> https://docs.docker.com/reference/cli/dockerd/#configure-cgroup-driver

将`/etc/docker/daemon.json`改成这样并重启守护进程:
```bash
[baka@centos7-minikube ~]$ cat /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
[baka@centos7-minikube ~]$ sudo systemctl daemon-reload
[baka@centos7-minikube ~]$ sudo systemctl restart docker
```

理论上改dockerd的unit也行，懒得试了

## 工作节点的cri为cri-dockerd时，且集群cni为flannel时使用kubeadm加入集群

> 本节参考:
>
> [kubeadm join](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-join/)

需要多加`--cri-socket unix:///var/run/cri-dockerd.sock`参数:
```bash
[baka@centos7-minikube ~]$ sudo kubeadm join 192.168.56.90:6443 --token f6xbzx.e73ojlyd6tl5eiqt --discovery-token-ca-cert-hash sha256:c7aa5c9a4f6e7a7813d9e7b7dd805197a1b7dc6b04b1579ba9857d1ddca683fa --cri-socket unix:///var/run/cri-dockerd.sock
```

此外加入后还需要修改改工作节点的podCIDR才能让flannel插件正常工作:
```bash
[baka@alma9-minikube ~]$ kubectl edit nodes centos7-minikube
```

如下:
```yaml
apiVersion: v1
kind: Node
metadata:
  annotations:
......
spec:
  podCIDR: 10.244.0.0/16
......
```

后续在master节点检查pods和node的状态:
```bash
[baka@alma9-minikube ~]$ kubectl get nodes
NAME               STATUS   ROLES           AGE     VERSION
alma9-minikube     Ready    control-plane   7d17h   v1.30.3
centos7-minikube   Ready    <none>          24m     v1.30.3
[baka@alma9-minikube ~]$ kubectl get pods -A
NAMESPACE      NAME                                     READY   STATUS    RESTARTS        AGE
kube-flannel   kube-flannel-ds-wmk5s                    1/1     Running   6 (120m ago)    6d21h
kube-flannel   kube-flannel-ds-wrccw                    1/1     Running   3 (6m58s ago)   24m
kube-system    coredns-7db6d8ff4d-ffnbr                 1/1     Running   4 (121m ago)    6d21h
kube-system    coredns-7db6d8ff4d-t6njg                 1/1     Running   4 (121m ago)    6d21h
kube-system    etcd-alma9-minikube                      1/1     Running   24 (121m ago)   7d17h
kube-system    kube-apiserver-alma9-minikube            1/1     Running   27 (121m ago)   7d17h
kube-system    kube-controller-manager-alma9-minikube   1/1     Running   4 (121m ago)    6d22h
kube-system    kube-proxy-b468x                         1/1     Running   5 (121m ago)    7d17h
kube-system    kube-proxy-mz76g                         1/1     Running   0               24m
kube-system    kube-scheduler-alma9-minikube            1/1     Running   26 (121m ago)   7d17h
```