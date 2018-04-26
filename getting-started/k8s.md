<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Kubeadm 安装 k8s](#kubeadm-%E5%AE%89%E8%A3%85-k8s)
  - [系统配置](#%E7%B3%BB%E7%BB%9F%E9%85%8D%E7%BD%AE)
  - [安装 docker](#%E5%AE%89%E8%A3%85-docker)
  - [镜像准备](#%E9%95%9C%E5%83%8F%E5%87%86%E5%A4%87)
  - [安装 kubeadm、kubelet、kubectl](#%E5%AE%89%E8%A3%85-kubeadmkubeletkubectl)
  - [配置 kubelet](#%E9%85%8D%E7%BD%AE-kubelet)
  - [集群安装](#%E9%9B%86%E7%BE%A4%E5%AE%89%E8%A3%85)
    - [集群部署](#%E9%9B%86%E7%BE%A4%E9%83%A8%E7%BD%B2)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Kubeadm 安装 k8s

## 系统配置
 
centos7 机器， 修改 `/etc/hosts`，修改 IP
 
```bash
$ cat /etc/hosts
192.168.20.103 master1
```

如果各个主机启用了防火墙，需要开放 k8s 各个组件所需要的端口，可以查看 [Installing kubeadm](https://kubernetes.io/docs/setup/independent/install-kubeadm/) 中的 `Check required ports` 一节。 
这里简单起见直接禁用防火墙：

```bash
systemctl stop firewalld
systemctl disable firewalld
```

禁用SELINUX：

```bash
$ setenforce 0
$ cat /etc/selinux/config
SELINUX=disabled
```

创建 `/etc/sysctl.d/k8s.conf` 文件，添加如下内容：

```txt
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

执行命令使修改生效:

```bash
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
```

由于 k8s 官方的 yum 源是需要科学上网的，在此，我们指定阿里的 yum 源：

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

## 安装 docker

```bash
yum install -y docker
systemctl enable docker && systemctl start docker
```

## 镜像准备

如果不能科学上网，可以使用我预先准备的[镜像](https://pan.baidu.com/s/1Zcw6d-tDyMMtfJrp_mzqdw), 然后使用以下命令载入镜像：

```bash
docker load -i k8s-1.10-images.tar.gz
```

## 安装 kubeadm、kubelet、kubectl

```bash
yum makecache fast && yum install -y kubelet kubeadm kubectl
```

## 配置 kubelet

安装完成后，我们还需要对kubelet进行配置，保证 kubelet 的配置文件参数 `--cgroup-driver` 与 docker 的 `Cgroup Driver` **一致**才行，我们可以通过docker info命令查看：

```bash
$ docker info |grep Cgroup
Cgroup Driver: cgroupfs
```

修改文件 kubelet 的配置文件 `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`，将其中的 KUBELET_CGROUP_ARGS 参数更改成cgroupfs：

```txt
Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"
```

k8s 从1.8开始要求关闭系统的 Swap ，如果不关闭，默认配置的 kubelet 将无法启动，我们可以通过 kubelet 的启动参数 `--fail-swap-on=false` 更改这个限制，所以我们需要在上面的配置文件中增加一项配置(**在ExecStart之前**)：

```txt
Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false"
```

重新加载配置文件

```bash
$ systemctl daemon-reload
```

## 集群安装

### 集群部署

使用kubeadm初始化集群，选择 master1 作为 master 节点，在 master1 执行下面的命令，修改 `--apiserver-advertise-address` 为你的机器 IP：

```bash
$ kubeadm init --kubernetes-version=v1.10.0 --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=1.2.3.4 --ignore-preflight-errors=Swap
```

配置使用 kubectl 访问集群的方式：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

查看一下集群状态, 确认个组件都处于 Healthy 状态：

```bash
$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}

$ kubectl get nodes
NAME       STATUS    ROLES     AGE       VERSION
c420v103   Ready     master    5m        v1.10.1
```
至此，k8s 集群部署成功。

集群初始化如果遇到问题，可以使用下面的命令进行清理：

```bash
kubeadm reset
ifconfig cni0 down
ip link delete cni0
ifconfig flannel.1 down
ip link delete flannel.1
rm -rf /var/lib/cni/
```
