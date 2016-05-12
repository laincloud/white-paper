# 在青云上安装 Lain

## 机器选型

- VPC ，关闭 DHCP
- 使用 centos7.2 镜像创建 lain-01  节点
- 修改 hostname。如 `hostname lain-01`(重新登录)
- 编辑网络配置，写静态 ip 和 DNS。
  ```sh
  # @lain-01 
  # vi /etc/sysconfig/network-scripts/ifcfg-eth0

  DEVICE=eth0
  BOOTPROTO=static
  TYPE=Ethernet
  NAME="System etho0"
  BROADCAST=192.168.200.255
  HWADDR=52:54:44:40:62:CC
  IPADDR=192.168.200.21 # IP
  IPV6INIT=yes
  IPV6_AUTOCONF=yes
  NETMASK=255.255.255.0
  NETWORK=192.168.200.1
  ONBOOT=yes
  DNS1=10.16.40.3
  DNS2=1.2.4.8
  DOMAIN=pek2.qingcloud.com
  GATEWAY=192.168.200.1
  ```
  然后重启 network，并测试。
  ```sh
  # @lain-01 
  service network restart # 重启
  curl www.douban.com # 测试一下是否通网
  ```

## Bootstrap

```sh
# @lain-01
git clone git@github.com/laincloud/lain.git lain
cd lain
# DOMAIN 默认值是 lain.local
./bootstrap -r registry.aliyuncs.com/laincloud --vip 192.168.200.201 --domain DOMAIN
```

## 配置 VPC 的公网路由器，端口转发

转发公网ip的 80/443 端口到 `192.168.200.201` 这个 lain vip 的 80/443

## Add node (optional)

同 bootstrap 前设置 lain-01 一样，需要设置网络:

```sh
# @lain-02
# vi /etc/sysconfig/network-scripts/ifcfg-eth0     写静态 ip 和 DNS

DEVICE=eth0
BOOTPROTO=static
TYPE=Ethernet
NAME="System etho0"
BROADCAST=192.168.200.255
HWADDR=52:54:34:96:76:55
IPADDR=192.168.200.22  # IP
IPV6INIT=yes
IPV6_AUTOCONF=yes
NETMASK=255.255.255.0
NETWORK=192.168.200.1
ONBOOT=yes
DNS1=10.16.40.3
DNS2=1.2.4.8
DOMAIN=pek2.qingcloud.com
GATEWAY=192.168.200.1
```
然后:
```sh
# @lain-02
service network restart
curl www.douban.com # 测试一下是否通网
```

```sh
# @lain-01
lainctl node add -p playbooks -q lain-02:192.168.200.22
```

> **注意**: 如果出现 ssh-copy-id  失败，可能需要把 `lain-01:/root/.ssh/lain.pub` 内容放到 `lain-02:/root/.ssh/authorized_keys`里，新增一行。
> 当然原因可能是多样的，最有可能就是 lain-02 的 `/root/.ssh` 目录或者目录中的文件权限不对


## Scale webrouter (optional)

参见 [webrouter scale](../maintain/webrouter.html#scale)
