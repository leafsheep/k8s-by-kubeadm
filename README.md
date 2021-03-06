# k8s by kubeadm 国内网络环境搭建指南

本教程简要阐述了使用kubeadm在国内网络环境搭建单主k8s集群的方法。

欢迎各种形式的建议、勘误及贡献。

Happy Hacking!

# 开始搭建

本教程使用的大部分bash脚本可以在`script`文件夹中找到。

在使用脚本之前先通读本`README.md`，避免翻车。

如果遇到问题，可以先查阅[常见问题和解决方案](https://github.com/nanmu42/k8s-by-kubeadm/issues?utf8=%E2%9C%93&q=label%3AQA+)。

## 先决条件

### 实例

* 一个或更多运行Ubuntu 16.04+/CentOS 7/Debian 9，2 GB以上内存，2核以上CPU的实例；
* 实例之间有网络联通；
* 确保每个实例有唯一的`hostname`, `MAC address`以及`product_uuid`（这个条件一般都能满足）：

```bash
# 查询MAC地址
ip link

# 查询 product_uuid
sudo cat /sys/class/dmi/id/product_uuid
```

### hostname

k8s会使用实例的hostname作为节点名称，因此有必要为每个实例取一个描述性较好的名称。

实例的`hostname`需要满足[DNS-1123](https://tools.ietf.org/html/rfc1123)规范：

* 字符集：数字、小写字母、`.`、`-`
* 以小写字母开头和结尾

正则表达式为：

```regexp
[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*
```

修改`hostname`方式如下（`script/01_change_hostname.sh`）：

```bash
# 将 master.localdomain 换为合适的值
echo "master.localdomain" > /etc/hostname 
echo "127.0.0.1   master.localdomain" >> /etc/hosts
echo "::1   master.localdomain" >> /etc/hosts
# 不重启的情况下使修改生效
sysctl kernel.hostname=master.localdomain
```

### 禁用Swap

`kubelet`要求宿主实例的交换空间（Swap）禁用以正常工作。

```bash
# 查看实例的swap设备
# 如果没有输出，说明没有启用swap，可略过余下步骤
cat /proc/swaps

# 关闭swap
swapoff -a

# 清理相应的注册项
nano /etc/fstab
```

## 设置安全组

云上实例需要放行安全组中的下列指定TCP入方向（这里假设安全组的出方向TCP/UDP全部放行）：

* 主节点（Master）
  * 6443
  * 2379-2380
  * 10250-10252
* 从节点（Worker）
  * 10250
  * 30000-32767

以上为Kubernetes本身需要开放的端口。

**注意**，网络插件（CNI，容器网络接口）另有需要开放的端口，本教程使用Flannel（`vxlan`模式）作为CNI，需要额外放行下列入方向端口：

* UDP 8472

## 安装容器运行时

本教程使用Docker作为容器运行时，请参阅[这里](https://docs.docker.com/v17.12/install/#server)进行安装。

## 安装kubeadm, kubelet 和 kubectl

由于一些原因，官方源无法在国内使用，这里使用国内镜像进行安装：

* Ubuntu（`script/02_install_kubeadm_ubuntu.sh`）

```bash
# 使用阿里云镜像
apt-get update && apt-get install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# 配置命令自动完成
echo "source <(kubectl completion bash)">> ~/.bashrc
echo "source <(kubeadm completion bash)">> ~/.bashrc
```

* CentOS（`script/02_install_kubeadm_centos.sh`）

```bash
# 使用阿里云镜像
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet

# 配置命令自动完成
echo "source <(kubectl completion bash)">> ~/.bashrc
echo "source <(kubeadm completion bash)">> ~/.bashrc
```

## 启动主节点，启动集群

选定一个实例作为主节点，运行下列命令（`script/03_boot_master.sh`）：

```bash
# Pass bridged IPv4 traffic to iptables’ chains. This is a requirement for some CNI plugins to work
sysctl net.bridge.bridge-nf-call-iptables=1

# flannel 要求指定该 pod-network-cidr
# 指定 image-repository 以使用国内镜像
kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers

# 等待片刻，让系统准备好
echo 'sleep a while for k8s to get ready...'
sleep 15
```

如果一切无误，kubeadm最后会有形如以下的输出：

```
kubeadm join 192.168.100.200:6443 --token some_token_here \
    --discovery-token-ca-cert-hash sha256:some_hash_here
```

记录上述输出，供从节点启动使用。

以一般用户运行下列命令，配置主节点所在实例的kubectl（`script/04_config_kubectl.sh`）：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 配置CNI

我们使用flannel作为CNI（`script/05_deploy_flannel.sh`）：

```bash
# 部署 flannel 作为 CNI
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
```

## （可选）让主节点也可以运行Pod

Kubernetes默认不在主节点上运行Pod，这里可以让调度器不再遵从这个策略。

这会提高资源利用率，代价是会降低主节点的安全性。

```bash
# （可选）让主节点上也能运行pod
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## 启动从节点，加入集群

在要作为从节点加入集群的实例上，运行上个步骤kubeadm的输出的加入命令：

```bash
kubeadm join 192.168.100.200:6443 --token some_token_here \
    --discovery-token-ca-cert-hash sha256:some_hash_here
```

## 检查节点状况

```bash
$ kubectl get node
```

```
NAME                 STATUS   ROLES    AGE     VERSION
master.localdomain   Ready    master   2m   v1.14.0
worker.localdomain   Ready    <none>   2m   v1.14.0
```

如果列表中几个节点状态都为`Ready`，那么恭喜，你成功完成了本教程，部署了一个单主节点的k8s集群！

## 本教程的可执行脚本

可执行脚本在本仓库的`script`文件夹中，使用前请阅读`script/README.md`。

## 下一步？

[官方文档](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#optional-controlling-your-cluster-from-machines-other-than-the-master)是下一个不错的起点，祝你好运！

# 参考文献/致谢

* [阿里巴巴开源镜像站](https://opsx.alibaba.com/mirror)
* https://kubernetes.io/docs/setup/independent/install-kubeadm/#check-required-ports
* https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
* https://serverfault.com/questions/684771/best-way-to-disable-swap-in-linux
* https://github.com/coreos/flannel/blob/master/Documentation/backends.md#recommended-backends
* https://blog.csdn.net/aixiaoyang168/article/details/78411511
* https://zhuanlan.zhihu.com/p/46341911
* https://kubernetes.feisky.xyz/ （很棒的学习资源）

# License

MIT License
