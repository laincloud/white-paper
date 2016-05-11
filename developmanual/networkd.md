# Networkd

## 介绍

1. networkd 是 LAIN 的 Layer 0 组件，每个 LAIN 节点都会运行，主要工作是负责网络相关的配置工作，包括 Iptables，IP，Dnsmasq 等。
1. 采用 Go 语言开发，Godep 管理依赖，使用 docker image 分发。
1. 主要流程是监听 Etcd/Lainlet/Container 的变化，触发对应的操作。

## 特性

### Virtual IP
1. 跟随 container 迁移的 IP，保证使用某固定 IP 可以一直访问到某 App 的某 Proc。
1. 配置: `/lain/config/vips/IP`, [配置说明](https://github.com/laincloud/networkd/blob/master/README.md)

### Node IP
1. 某些云主机不能自己设置 IP，这种情况如果需要对外暴露 IP，就只能使用 Node IP 了。
1. 配置: `/lain/config/vips/IP:PORT`, [配置说明](https://github.com/laincloud/networkd/blob/master/README.md)

### Dnsmasq
1. 监听 Lainlet config 配置，设置 dnsmasq 的 hosts 以及 servers
    1. `/etc/dnsmasq.hosts`
    1. `/etc/dnsmasq.servers` Dnsmasq 的 upstream server
1. upstream server 是对应的 Tinydns app

### Resolv.conf
1. 确保 Node 优先从 Dnsmasq 查询 DNS 记录

### Swarm
1. 配置 Dnsmasq 的 `swarm.lain` 记录指向 swarm-manger 所在 Node IP

### Tinydns
1. 配置 Dnsmasq 的 servers 指向 Tinydns app 所在 Node IP

### Webrouter
1. 配置 Tinydns 的 `webrouter.lain` 记录指向 Webrouter app 所在 Node IP (在云主机不支持设置 VIP 的情况下)

## 构建

1. 参考: [Dockerfile](https://github.com/laincloud/networkd-binary/blob/master/dockerfiles/networkd.df)
