# Google Kubernetes安装手册

用于容器编排，服务治理等。而和Istio配合，实现更前沿的Service Mesh（服务网络）。Kubernetes + Istio是目前的事实标准，不过rancher的k3s已入选CNCF，微软的OSM也已公布，一切还不好说。

~~不得不说Kubernetes的文档真的是一言难尽，看个文档跟看《信条》似的(￣▽￣|||)。~~

## 1. 前提条件

### 1.1. 高可用主机/虚拟机数量要求

高可用场合，需要至少3台主机/虚拟机用（以下简称机器）作主节点（Master Node），至少3台机器用作工作节点（Work Node）。

### 1.2. 操作系统要求

- [ ] Ubuntu 16.04+
- [ ] Debian 9+
- [x] CentOS 7
- [ ] Red Hat Enterprise Linux (RHEL) 7
- [ ] Fedora 25+
- [ ] HypriotOS v1.0.1+
- [ ] Container Linux (已测试于2512.3.0版本)

### 1.3. 内存要求

每台机器2GB或更多的RAM（如果少于这个数值将会影响应用的运行内存）。

### 1.4. CPU核心数要求

每台机器2核或更多的CPU核心数。

### 1.5. 网络要求

集群中的所有机器的网络彼此均能相互连接(公网和内网均可)。

#### 1.5.1. 端口

确保开启机器上的以下端口：

主节点用

协议|方向|端口范围|作用|使用者
:--:|:--:|--|--|--
TCP|入站|6443|Kubernetes API Server|所有组件
TCP|入站|2379-2380|etcd server client API|kube-apiserver、etcd
TCP|入站|10250|Kubelet API|kubelet自身、Control plane组件
TCP|入站|10251|kube-scheduler|kube-scheduler自身
TCP|入站|10252|kube-controller-manager|kube-controller-manager自身

工作节点用

协议|方向|端口范围|作用|使用者
:--:|:--:|--|--|--
TCP|入站|10250|Kubelet API|kubelet自身、Control plane组件
TCP|入站|30000-32767|NodePort服务|所有组件

虽然控制平面节点已经包含了etcd的端口，你也可以使用自定义的外部etcd集群，或是指定自定义端口。

你使用的pod网络插件(见下) 也可能需要某些特定端口开启。由于各个pod网络插件都有所不同，请参阅其各自文档中对端口的要求。

可通过以下命令查看、打开、关闭端口：

- 查看端口状况

  ``` bash
  netstat -anp
  ```

- 打开端口（替换`<portNumber>`为所需打开端口号）

  ``` bash
  iptables -A INPUT -p tcp --dport <portNumber> -j ACCEPT
  ```

- 关闭端口（替换`<portNumber>`为所需关闭端口号）

  ``` bash
  iptables -A OUTPUT -p tcp --dport <portNumber> -j DROP
  ```

- 保存设置

  ``` bash
  service iptables save
  ```

#### 1.5.2. hostname，MAC地址和product_uuid

确保每个节点上的hostname，MAC地址和product_uuid的唯一性。一般来讲，硬件设备会拥有唯一的地址，但是有些虚拟机的地址可能会重复。Kubernetes使用这些值来唯一确定集群中的节点。如果这些值在每个节点上不唯一，可能会导致安装失败。

- 获取网络接口的MAC地址

  ``` bash
  ip link
  ```

  或者

  ``` bash
  ifconfig -a
  ```

- 校验product_uuid

  ``` bash
  sudo cat /sys/class/dmi/id/product_uuid
  ```

#### 1.5.3. 网络适配器

如果存在一个以上的网络适配器，同时Kubernetes组件通过默认路由不可达的场合，建议预先添加IP路由规则，以便Kubernetes集群可以通过对应的适配器完成连接。

#### 1.5.4. 防火墙

确保`iptables`工具能够处理bridged traffic。

执行`lsmod | grep br_netfilter`命令，确保`br_netfilter`模块已加载；或者通过`sudo modprobe br_netfilter`命令显式加载。

然后，确保`sysctl`的配置文件中，`net.bridge.bridge-nf-call-iptables`被设置为`1`，例如：

``` bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

请参阅[网络插件要求](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#network-plugin-requirements)以获取更多的信息。

### 1.6. 其他要求

#### 1.6.1. SWAP（交换分区）

**务必**禁用swap，以确保kubelet工作正常。

1. 查看/etc/fstab

    ``` bash
    more /etc/fstab
    ```

1. 找到swap分区的记录，把加载swap分区的那行记录注释掉（使用`#`），例如：

    ``` text
    #/dev/mapper/cl-swap     swap                    swap    defaults        0 0
    ```

1. 重启主机/虚拟机

    ``` bash
    systemctl reboot
    ```

1. 使用`free -m`命令确认，显示结果为应为：

    ``` bash
                  total        used        free      shared  buff/cache   available
    Swap:             0           0           0
    ```

## 2. 安装

### 2.1. 在线安装

#### 2.1.1. 安装运行时

为了能让容器在Pods上运行，需要安装CRI（Container Runtime Interface，容器运行时接口）。

Kubernetes默认使用CRI与你指定的容器运行时交互；如果未指定，则kubeadm将通过扫描已知的UNIX域套接字来自动检测已安装的容器运行时。下表列出了可监测到的容器运行时以及相关的域套接字：

运行时|域套接字
--|--
Docker|/var/run/docker.sock
containerd|/run/containerd/containerd.sock
CRI-O|/var/run/crio/crio.sock

如果同时检测到docker和containerd，则优先选择docker。这是因为docker 18.09附带了containerd并且两者都是可以检测到的。如果检测到另外两个及以上运行时，kubeadm将报错退出。

kubelet通过Docker内建的`dockershim`CRI实现与之集成。

请以`root`身份执行本指南中的所有命令。例如，在命令前附加`sudo`前缀，或者成为`root`并以该用户身份运行命令。

##### 2.1.1.1. Cgroup驱动

更改设置，令容器运行时和kubelet使用`systemd`作为cgroup驱动，以此使系统更为稳定。请注意在docker下设置`native.cgroupdriver=systemd`选项。

> 注意：
>
> 强烈建议不要更改已加入集群的节点的cgroup驱动。如果kubelet已经使用某cgroup驱动的语义创建了pod，尝试更改运行时以使用别的cgroup驱动，为现有Pods重新创建PodSandbox时会产生错误。重启kubelet也可能无法解决此类问题。推荐将工作负载逐出节点，之后将节点从集群中删除并重新加入。

2.1.1.2.~2.1.1.4.为3种可安装的容器运行时，请选择一种进行安装，其中除CRI-O外均可离线安装。

##### 2.1.1.2. Docker

推荐安装19.03.11版本，不过1.13.1、17.03、17.06、17.09、18.06和18.09版本也是可以的。

使用以下命令在系统上安装Docker：

``` bash
# 安装Docker CE
## 设置仓库
### 安装所需包
yum install -y yum-utils device-mapper-persistent-data lvm2
```

``` bash
## 新增Docker仓库
yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
```

``` bash
# 安装Docker CE
yum update -y && yum install -y \
  containerd.io-1.2.13 \
  docker-ce-19.03.11 \
  docker-ce-cli-19.03.11
```

``` bash
## 创建/etc/docker目录
mkdir /etc/docker
```

``` bash
# 设置Docker daemon
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
```

``` bash
mkdir -p /etc/systemd/system/docker.service.d
```

``` bash
# 重启Docker
systemctl daemon-reload
systemctl restart docker
```

如果你希望开机即启动docker服务，执行以下命令：

``` bash
sudo systemctl enable docker
```

请参阅[官方Docker安装指南](https://docs.docker.com/engine/install/)以获取更多的信息。

##### 2.1.1.3. CRI-O

使用以下命令在系统中安装CRI-O：

``` bash
# 准备环境
modprobe overlay
modprobe br_netfilter

## 设置必需的sysctl参数，这些参数在重启后仍然存在
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
```

``` bash
# 安装CRI-O
## 操作系统$OS
## Centos 8            CentOS_8
## Centos 8 Stream     CentOS_8_Stream
## Centos 7            CentOS_7
### CRI-O版本$VERSION，与Kubernetes相同
### Kubernetes 1.19    1.19
### Kubernetes 1.19.1  1.19:1.19.1
OS=CentOS_7
VERSION=1.19:1.19.1

curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
yum install cri-o
```

``` bash
# 启动CRI-O
systemctl daemon-reload
systemctl start crio
```

请参阅[CRI-O安装指南](https://github.com/cri-o/cri-o#getting-started)来获取更多的信息。

##### 2.1.1.4. containerd

使用以下命令在系统上安装容器：

``` bash
# 准备环境
cat > /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

## 设置必需的sysctl参数，这些参数在重新启动后仍然存在
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
```

``` bash
# 安装containerd
## 设置仓库
### 安装所需包
yum install -y yum-utils device-mapper-persistent-data lvm2
```

``` bash
## 新增Docker仓库
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

``` bash
## 安装containerd
yum update -y && yum install -y containerd.io
```

``` bash
## 配置containerd
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

``` bash
# 重启containerd
systemctl restart containerd
```

systemd

要使用`systemd`cgroup驱动，请在`/etc/containerd/config.toml`中设置：

``` bash
[plugins.cri]
systemd_cgroup = true
```

当使用kubeadm时，请手动配置[kubelet的cgroup驱动](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#configure-cgroup-driver-used-by-kubelet-on-control-plane-node)。

#### 2.1.2. 安装kubeadm、kubelet和kubectl

需要在每台机器上安装以下的软件包：

- `kubeadm`：用于引导集群的指令。
- `kubelet`：工作节点必须安装。在集群中的每个节点上运行的组件，用于启动（一个或多个）pod和容器等。
- `kubectl`：主节点必须安装。用于与集群通信的命令行工具，通过kubectl可以部署应用，查看和管理集群资源，并且查看日志。

kubeadm*不会*为你安装或者管理kubelet或kubectl，所以你须要确保它们与通过kubeadm安装的控制平面的版本相匹配。否则，存在发生版本偏差的风险，这可能会导致一些预料之外的错误和问题。不过，控制平面与kubelet间的相差一个次要版本是受支持的，只不过kubelet的版本不可以超过API服务器的版本。例如，1.7.0版本的kubelet可以完全兼容1.8.0版本的API服务器，反之则不可以。

关于版本偏差的更多信息，请参阅以下文档：

- Kubernetes[版本与版本间的偏差策略](https://kubernetes.io/docs/setup/release/version-skew-policy/)
- Kubeadm-specific[版本偏差策略](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#version-skew-policy)

##### 2.1.2.1. 安装kubeadm、kubelet和kubectl

可通过以下命令安装`kubeadm`，`kubelet`和`kubectl`：

``` bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 将SELinux设置为permissive（宽容）模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```

> 注意：
>
> - 通过运行命令`setenforce 0`和`sed ...`将SELinux设置为宽容模式可以有效的将其禁用。这是允许容器访问主机文件系统所必须的，例如pod网络就需要这样做。 在kubelet对SELinux的支持得到改进前，不得不这么做。
> - 如果你知道如何配置SELinux，也可以保持SELinux处于启用状态，不过可能需要一些kubeadm不支持的设置。
> - TODO repo uri要替换

kubelet现在每隔几秒就会重启，陷入了一个等待kubeadm发布指令的死循环。这是正常现象， 初始化控制平面后，kubelet将正常运行。

##### 2.1.2.2. 配置kubelet

在使用Docker时，kubeadm会自动为kubelet检测cgroup驱动并在运行期间对`/var/lib/kubelet/config.yaml`文件进行配置。

如果你使用不同的CRI，须将你的`cgroupDriver`传递给`kubeadm init`，像这样：

``` yml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: <value>
```

请参阅[结合一份配置文件来使用`kubeadm init`](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file)来获取更多的信息。

> 注意：
>
> **只有**在CRI的cgroup驱动程序不是`cgroupfs`时才必须这样做，因为它已经是kubelet中的默认值。

必须重新启动kubelet：

``` bash
systemctl daemon-reload
systemctl restart kubelet
```

对其他容器运行时（比如CRI-O和containerd）的cgroup驱动自动检测功能正在开发中。

##### 2.1.2.3. 配置kubectl（可选）

启用 shell 自动补全功能。kubectl为Bash和Zsh提供了自动补全支持，可以节省大量打字时间。下面以Bash为例设置自动补齐（Zsh可参阅[这里](https://kubernetes.io/docs/tasks/tools/install-kubectl/#enabling-shell-autocompletion)）：

``` bash
# 1. 确认bash-completion依赖是否已安装
## 如果执行成功，直接启用kubectl自动补齐即可
### 如果失败，先安装bash-completion
type _init_completion

# 2. 安装bash-completion
yum install bash-completion

# 3. 检查安装（须先重新加载shell）
## 确认/usr/share/bash-completion/bash_completion是否已创建
type _init_completion
### 如果执行成功，启用kubectl自动补齐
### 否则执行以下命令并重新加载shell
source /usr/share/bash-completion/bash_completion

# 4. 启用kubectl自动补齐（两种方案等价）
## 方案1：在 ~/.bashrc 文件中源引自动补齐脚本
echo 'source <(kubectl completion bash)' >>~/.bashrc
## 方案2：将自动补齐脚本添加到目录 /etc/bash_completion.d
kubectl completion bash >/etc/bash_completion.d/kubectl
```

#### 2.1.3. 安装Pod网络附加组

#### 2.1.4. 使用kubeadm创建集群

### 2.2. 离线安装

> TODO: 需要使用PXE

## 3. 附录

### 3.1. `kubeadm init`命令参数一览

### 3.2. `kubeadm init`命令配置文件选项一览

### 3.3. 使用kubeadm定制控制平面配置

### 3.4. 单独安装kubectl

#### 3.4.1. 安装kubectl

可通过以下命令安装`kubectl`：

``` bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubectl
```

#### 3.4.2. 验证kubectl配置

kubectl需要一个kubeconfig配置文件以发现并访问Kubernetes集群。当使用[kube-up.sh](https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh)脚本创建Kubernetes集群或者成功部署Minikube集群后，会自动生成[kubeconfig](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)配置文件。默认情况下，kubectl配置文件在`~/.kube/config`。

可通过获取集群状态检查kubectl是否被正确配置：

``` bash
kubectl cluster-info
```

如果看到一个URL被返回，那么kubectl已经被正确配置，能够正常访问Kubernetes集群。

如果看到类似以下的信息被返回，那么说明kubectl没有被正确配置，或者无法正常访问Kubernetes集群。

``` text
The connection to the server <server-name:port> was refused - did you specify the right host or port?
```

例如，如果你打算在笔记本电脑（本地）上运行Kubernetes集群，则需要首先安装Minikube之类的工具，然后重新运行上述命令。

如果kubectl的cluster-info能够返回URL响应，但无法访问集群，可以使用下面的命令检查配置是否正确：

``` bash
kubectl cluster-info dump
```

> 参考文献：
>
> - <https://k8smeetup.github.io/docs/home/>
> - <http://docs.kubernetes.org.cn/457.html>
> - <https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh>