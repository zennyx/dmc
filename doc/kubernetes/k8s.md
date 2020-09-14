# Google Kubernetes安装手册

用于容器编排，服务治理等。而和Istio配合，实现更前沿的Service Mesh（服务网络）。Kubernetes + Istio是目前的事实标准，不过rancher的k3s已入选CNCF，微软的OSM也已公布，一切还不好说。

~~不得不说Kubernetes的文档真的是一言难尽，看个文档跟看《信条》似的(￣▽￣|||)。~~

## 1. 前提条件

### 1.1. 高可用主机/虚拟机数量要求

高可用时，需要至少3台主机/虚拟机用（以下简称机器）作主节点（Master Node），至少3台机器用作工作节点（Work Node）。

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

#### 1.5.2. MAC地址和product_uuid

确保每个节点上MAC地址和product_uuid的唯一性。一般来讲，硬件设备会拥有唯一的地址，但是有些虚拟机的地址可能会重复。Kubernetes使用这些值来唯一确定集群中的节点。如果这些值在每个节点上不唯一，可能会导致安装失败。

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

#### 2.1.2. 安装kubeadm、kubelet和kubectl

#### 2.1.3. 安装Pod网络附加组

#### 2.1.4. 使用kubeadm创建集群

### 2.2. 离线安装

## 3. 附录

### 3.1. `kubeadm init`命令参数一览

### 3.2. `kubeadm init`命令配置文件选项一览

### 3.3. 使用kubeadm定制控制平面配置