# LAIN 的快速安装

## Vagrant

### 环境依赖

* Linux / MacOS
* 能够连接到互联网
* VirtualBox
* Vagrant
* 最少 2G 剩余内存（如果需要拉起多个节点，最少 3G）

### 初始化

#### 获取代码

你可以从 GitHub 上获取到已经发布的 LAIN 版本：

https://github.com/laincloud/lain/releases

下载源码后在目标机器上解压即可。


#### 启动并进入第一个节点

```
vagrant up node1
vagrant ssh node1
```

耗时取决于 vagrant box 下载时间

#### 初始化第一个节点

```
[vagrant@node1 ~]$ sudo su -
[vagrant@node1 ~]# cd /vagrant
[vagrant@node1 vagrant]# ./bootstrap -r registry.aliyuncs.com/laincloud --vip=192.168.77.201
```

初始化需要至少 20 分钟，取决于网络速度

#### 添加更多节点

```
vagrant up node2

# 待 node2 启动后
[vagrant@node1 ~]$ cd /vagrant
[vagrant@node1 ~]$ sudo lainctl node add -p playbooks -q node2:192.168.77.22
# root 密码为 vagrant
```

同理可以如此添加 `node3`

## 物理服务器 / 虚拟机
### 环境依赖
* CentOS 7.2
* NTP 服务保证节点间时间一致
* 需要能访问到可用的 yum 源（包括 epel）
* 能够连接到互联网
* 各节点之间能够互相 ssh
* 各节点 hostname 不同
* 各个节点位于同一个路由器之内

### 初始化
#### 第一个节点

```
git clone https://github.com/laincloud/lain.git
cd lain
# 选择一个同网段的未被使用的 IP 地址作为 VIP
sudo ./bootstrap -r registry.aliyuncs.com/laincloud --vip={{ vip }}
```

#### 添加更多节点
```
# 需要输入 root 密码
sudo lainctl node add -p playbooks -q {{ hostname }}:{{ hostname }} 
```

## 云服务器
### 环境依赖
* CentOS 7.2
* NTP 服务保证节点间时间一致
* 需要能访问到可用的 yum 源（包括 epel）
* 能够连接到互联网
* 各节点之间能够互相 ssh
* 各节点 hostname 不同
* 各个节点位于同一个 VPC （或虚拟路由器）之内

### 初始化

#### 第一个节点
```
git clone https://github.com/laincloud/lain.git
cd lain

# 如果 VPC 不对数据包进行来源 IP 限制（如青云）
sudo ./bootstrap -r registry.aliyuncs.com/laincloud

# 如果 VPC 限制了数据包的来源 IP（如阿里云）
sudo ./bootstrap -r registry.aliyuncs.com/laincloud --ipip

```

#### 添加更多节点

```
# 需要输入 root 密码
sudo lainctl node add -p playbooks -q {{ hostname }}:{{ hostname }} 
```

## FAQ

### add-node ssh-copy-id 失败
如果出现 ssh-copy-id  失败，可能需要把 `node1:/root/.ssh/lain.pub` 内容放到 `node2:/root/.ssh/authorized_keys`里，新增一行。当然原因可能是多样的，最有可能就是 lain-02 的 `/root/.ssh` 目录或者目录中的文件权限不对
