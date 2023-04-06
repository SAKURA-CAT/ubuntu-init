# ubuntu-init
> amd平台 Ubuntu 国内从单一镜像安装的流程

# 写在前面

受限于国内网络原因，乌班图的安装需要换源，介于每一次重装系统都要上网搜一些奇奇怪怪的资料，效率低下，所以不如自己维护一个文档来的实在。

* 考虑到后续可能更多是命令行部署，所以接下来更多会使用命令行进行一些相关操作，而非图形化界面
* 后续考虑会写为脚本形式，目前先写为markdown文字形式

## 使用平台

本仓库建立的背景是笔者在学习kubernets的相关知识的时候，需要创建多个虚拟机实现多机集群。所以笔者在配环境的时候使用了vmware workstation，版本是17。

### 乌班图版本

乌班图版本是**22.04**，并且在虚拟机安装的时候，选择了`安装所有需要的软件包和第三方软件`，所以暂时，这方面不在我的考虑范围内。

<img src="https://picture-bed-1305323352.cos.ap-beijing.myqcloud.com/uPic/20230405img_v2_f4cd2bcb-00d0-4f07-8ff8-173abbdbb08g.jpg" />

# 从这开始

接下来如果没有特殊说明，皆以非root用户（具体用户名称取决于个人的设置，笔者这里叫做`master`，取自k8s的master节点。

## 换源

一开始当然是换源，常用的Ubuntu版本代号如下：

- Ubuntu 22.04：jammy
- Ubuntu 20.04：focal
- Ubuntu 18.04：bionic
- Ubuntu 16.04：xenial

对于本版本而言选择jammy，如果要使用其他版本，直接把下面源中`jammy`改为其他的就行，我们这里的源选择阿里源：

```bash
sudo tee /etc/apt/sources.list <<-'EOF'
deb https://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb https://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
deb https://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
EOF
```

此外，如果不确定具体版本号，可以输入`sudo cat /etc/apt/sources.list`查看原始的源的内容进行确认。

换源完毕后执行 `sudo apt update` 即可查看是否换成功，如果没有报错说明换成功

## 安装必要包

必要的包包括：`net-tools`、`open-ssh`、`vim`等，持续更新...

### net-tools

这是一个网络管理包，`ifconfig`命令等都依赖于net-tools，运行以下命令即可完成安装：

```bash
sudo apt install net-tools
```

### openssh-server

一般是将乌班图用作服务器，所以需安装openssh-server：

```bash
sudo apt install openssh-server
```

安装完成后启动服务端：

```bash
sudo systemctl start sshd
```

可以验证是否启动：

```bash
sudo systemctl status sshd
```

如果服务端已成功启动，则会显示`active (running)`字样。

### vim

```bash
sudo apt install vim
```

## Docker与Kubernets安装

### 环境初始化

#### 关闭swap

需要关闭swap的几个原因有：

* Kubernetes对于内存的限制、请求和使用是基于cgroups实现的，而cgroups是依赖于内存的，因此在使用内存限制的情况下，Kubernetes无法保证容器内存使用的一致性，导致可能出现意外的行为。

* 内存swap区的使用会影响容器的性能，当容器在内存使用上限时，如果swap也被使用，就会导致系统出现大量的磁盘读写操作，从而影响容器的性能。
* 当内存swap区被使用时，如果物理内存达到极限，操作系统就会通过swap来释放内存，但这种情况下，操作系统可能会杀死进程，从而导致容器崩溃。

可以经过以下步骤禁用内存交换：

1. 通过以下命令检查是否存在交换分区和交换文件：

   ```bash
   sudo swapon --show
   ```

1. 通过以下命令关闭交换分区：

   ```bash
   sudo swapoff -a
   ```

2. 编辑 `/etc/fstab` 文件，注释掉所有与交换相关的行。例如，如果有一行包含 `/swapfile`，则可以将其注释掉：

   ```
   # /swapfile                none            swap    sw              0       0
   ```

3. 重启系统以应用更改。

> 如果使用云服务器或许无法用上述方式永久禁用swap，因为云提供商可能会自动将swap打开。在这种情况下，您需要查看云提供商的文档，以确定如何禁用swap

#### 开启ip转发

在网络中，IP转发是将一个数据包从一个网络设备（例如路由器）传输到另一个网络设备的过程。在Linux系统上，开启IP转发可以使主机作为路由器或网关，从而实现将数据包转发到其他网络设备。因此，在部署Kubernetes集群时，需要开启IP转发，以便于节点之间的网络通信和负载均衡。

可以使用以下命令开启ip转发：

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

> 该命令将net.ipv4.ip_forward内核参数设置为1，使Linux操作系统支持IP转发功能。

如果要永久开启IP转发，可以编辑` /etc/sysctl.conf`文件，找到以下行取消注释即可：

```yaml
#net.ipv4.ip_forward=1
```

### 安装Docker

需要注意的是，k8s并不需要指定docker版本。

1. 安装依赖

   ```bash
   sudo apt-get update
   sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
   ```

2. 安装gpg证书

   ```bash
   curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
   ```

3. 写入软件源信息

   ```bash
   sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
   ```

4. 更新并安装docker-ce和docker-compose

   ```bash
   sudo apt-get -y update
   sudo apt install -y docker-ce
   sudo apt install -y docker-compose
   ```

#### 其他设置

1. 将docker设置为开机自启

   ```bash
   sudo systemctl enable docker
   ```

2. 普通用户使用docker

   ```bash
   # 添加docker用户组
   sudo groupadd docker 
   # 将当前用户加入到docker用户组中
   sudo gpasswd -a $USER docker
   # 更新用户组
   newgrp docker
   ```

### 安装Kubernets

1. 添加证书

   ```bash
   curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add - 
   ```

2. 添加apt源

   ```bash
   sudo vim /etc/apt/sources.list.d/kubernetes.list
   # 写入内容
   deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
   ```

3. 更新并查看可安装版本

   ```bash
   sudo apt-get update
   apt-cache madison kubelet
   ```

4. 安装指定版本

   ```bash
   # 注意,因为V1.24以上的k8s版本已经弃用了docker，所以只能安装1.24以下版本，这里选择1.23.10
   sudo apt-get install -y kubelet=1.23.10-00 kubeadm=1.23.10-00 kubectl=1.23.10-00
   ```

#### 初始化一个k8s cluster

也就是初始化一个master节点，这里需要注意，国内`https://k8s.gcr.io/v2/`一般情况下是连不上的，得用阿里的镜像源：

```bash
 sudo kubeadm init --kubernetes-version=v1.23.10 \
 									 --apiserver-advertise-address 当前ip地址 \
                   --pod-network-cidr=10.244.0.0/16 \
                   --image-repository=registry.aliyuncs.com/google_containers \
                   --v 5
```

#### 其他设置

1. 设置开机启动

   ```bash
   sudo systemctl enable kubelet && sudo systemctl start kubelet
   ```

2. 修改docker的配置

   ```bash
   sudo vim /etc/docker/daemon.json
   # 写入以下内容:
   {
     "exec-opts": ["native.cgroupdriver=systemd"],
     "registry-mirrors": [
     "http://hub-mirror.c.163.com",
     "https://docker.mirrors.ustc.edu.cn",
     "https://registry.docker-cn.com"
     ]
   }
   sudo service docker restart
   ```

   `exec-opts`是用于设置 Docker 守护进程的 Cgroup 驱动程序为 Systemd。具体来说，它会在 `/etc/docker/daemon.json `文件中添加一个 "exec-opts" 键，其值为` ["native.cgroupdriver=systemd"]`，表示将 Cgroup 驱动程序设置为 Systemd。

   Cgroup 是 Linux 内核提供的一个机制，用于限制进程的资源使用，包括 CPU、内存、磁盘等。Docker 使用 Cgroup 来限制容器的资源使用。而 Systemd 是 Linux 下的一个初始化系统，它可以管理系统的进程、服务、挂载点等，并支持 Cgroup。

   通过将 Docker 的 Cgroup 驱动程序设置为 Systemd，可以更好地与宿主机的 Systemd 集成，提高容器的稳定性和可靠性。同时，也可以避免一些 Cgroup 相关的问题，如 CPU 和内存限制不准确等。

3. 普通用户使用kubectl（首先确保你已经切换到需要设置的用户

   ```bash
   mkdir -p $HOME/.kube
   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
   sudo chown $(id -u):$(id -g) $HOME/.kube/config
   # 开启自动补全
   echo "source <(kubectl completion bash)" >> ~/.bashrc
   ```

至此大功告成，后续的配置不在本部分的记录范围内
