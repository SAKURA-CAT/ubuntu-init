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
deb http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
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

## Docker与Kubernets安装

