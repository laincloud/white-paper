# 安装 LAIN 集群

### 本节包含三种场景下安装LAIN集群
* [本地启动虚拟机安装LAIN集群，可供本地测试开发](#1)
* [物理服务器/虚拟机安装LAIN集群](#2)
* [云服务器安装LAIN集群](#3)
-----------------------------

<h2 id="1">本地安装LAIN集群</h2>

### 环境依赖

* Linux / MacOS
* 能够连接到互联网
* VirtualBox 5.1.22 r115126 (Qt5.6.2)
* Vagrant 1.9.4
* 最少 2G 剩余内存（如果需要拉起多个节点，最少 3G）

### 初始化

#### 获取代码

你可以从 GitHub 上获取到已经发布的 LAIN 版本源码压缩包lain-VERSION.tar.gz：

https://github.com/laincloud/lain/releases

下载源码后在目标机器上解压即可。
```
tar zxvf lain-VERSION.tar.gz
```

#### 启动并进入第一个节点

```
cd lain-VERSION
vagrant up
vagrant ssh
```

耗时取决于 vagrant box 下载时间

> 如果出现以下错误：
>
> ```
> Vagrant was unable to mount VirtualBox shared folders. This is usually
> because the filesystem "vboxsf" is not available. This filesystem is
> made available via the VirtualBox Guest Additions and kernel module.
> Please verify that these guest additions are properly installed in the
> guest. This is not a bug in Vagrant and is usually caused by a faulty
> Vagrant box. For context, the command attempted was:
>
> mount -t vboxsf -o uid=1000,gid=1000 vagrant /vagrant
>
> The error output from the command was:
>
> /sbin/mount.vboxsf: mounting failed with the error: No such device
> ```
>
> 这个错误是因为宿主机的 Virtual Box 的 Guest Additions 与 `laincloud/centos-lain`
> box 已安装的 Guest Additions 版本不一致引起的，导致无法创建 `/vagrant` 这个同步
> 目录。请修改工程根目录下的 Vagrantfile，禁止宿主机强行安装新版本的 Guest Additions，
> 即添加如下配置：
>
> ```
> config.vbguest.auto_update = false
> ```

#### 初始化第一个节点
```
[vagrant@node1 ~]$ sudo su
[vagrant@node1 ~]# cd /vagrant
[vagrant@node1 vagrant]# ./bootstrap -r docker.io/laincloud --vip=192.168.77.201
```

初始化需要至少 20 分钟，取决于网络速度

> 国内用户建议通过 -m 参数使用 aliyun 的加速器下载镜像，使用方式为
> ```
> [vagrant@node1 vagrant]# ./bootstrap -m https://l2ohopf9.mirror.aliyuncs.com -r >docker.io/laincloud --vip=192.168.77.201
> ```

#### 添加更多节点

```
vagrant up node2

# 待 node2 启动后
[vagrant@node1 ~]$ cd /vagrant
[vagrant@node1 ~]$ sudo lainctl node add -p playbooks node2:192.168.77.22
# root 密码为 vagrant
```

#### 同理可以如此添加 `node3`
---------------

<h2 id="2">物理服务器/虚拟机安装LAIN集群</h2>

### 环境依赖
* CentOS 7.2
* NTP 服务保证节点间时间一致
* 需要能访问到可用的 yum 源（包括 epel）
* 能够连接到互联网
* 各节点之间能够互相 ssh
* 各节点 hostname 不同
* 各个节点位于同一个路由器之内

### 初始化
#### 获取代码

你可以从 GitHub 上获取到已经发布的 LAIN 版本源码压缩包lain-VERSION.tar.gz：

https://github.com/laincloud/lain/releases

下载源码后在目标机器上解压即可。
```
tar zxvf lain-VERSION.tar.gz
```

#### 第一个节点

```
cd lain-VERSION
# 选择一个同网段的未被使用的 IP 地址作为 VIP
sudo ./bootstrap -r docker.io/laincloud --vip={{ vip }}
```

#### 添加更多节点
```
# 需要输入 root 密码
sudo lainctl node add -p playbooks {{ hostname }}:{{ ip }} 
```
----------------------

<h2 id="3">云服务器安装LAIN集群</h2>

### 环境依赖

* CentOS 7.2
* NTP 服务保证节点间时间一致
* 需要能访问到可用的 yum 源（包括 epel）
* 能够连接到互联网
* 各节点之间能够互相 ssh
* 各节点 hostname 不同
* 各个节点位于同一个 VPC （或虚拟路由器）之内

### 初始化

#### 获取代码

你可以从 GitHub 上获取到已经发布的 LAIN 版本源码压缩包lain-VERSION.tar.gz：

https://github.com/laincloud/lain/releases

下载源码后在目标机器上解压即可。
```
tar zxvf lain-VERSION.tar.gz
```

#### 第一个节点
```
cd lain-VERSION

# 如果 VPC 不对数据包进行来源 IP 限制（如青云）
sudo ./bootstrap -r docker.io/laincloud

# 如果 VPC 限制了数据包的来源 IP（如阿里云）
sudo ./bootstrap -r docker.io/laincloud --ipip
```

#### 添加更多节点

```
# 需要输入 root 密码
sudo lainctl node add -p playbooks {{ hostname }}:{{ ip }} 
```
------------------------
## FAQ

### add-node ssh-copy-id 失败
如果出现 ssh-copy-id  失败，可能需要把 `node1:/root/.ssh/lain.pub` 内容放到 `node2:/root/.ssh/authorized_keys`里，新增一行。当然原因可能是多样的，最有可能就是 lain-02 的 `/root/.ssh` 目录或者目录中的文件权限不对

## 视频演示

本视频展示了集群的初始化、扩容过程。

视频地址：http://www.bilibili.com/video/av4671059/

> 详细的集群管理请见LAIN白皮书第四章：[集群管理员手册](../adminmanual/index.html)。
