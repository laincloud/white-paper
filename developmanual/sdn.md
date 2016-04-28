# Lain SDN

## 基本介绍

Lain 的容器间通信使用 Project Calico 提供的开源方案实现的，特点如下：

* 每个容器有自己的专有 ip（不同于服务器 ip），容器升级或迁移 ip 不变
* 同一个应用下的所有容器默认互相能够通信，不同应用的容器间默认不能通信
* 不依赖特殊硬件或网络环境，只要三层互通即可
* 使用 libnetwork 插件机制，和 Docker 结合的很好
* 转发平面使用 Linux 原生的路由和转发机制，没有封包解包，性能好
* 控制平面基于 Iptables，能做到精细控制，并且实现了安全组功能


## 整体设计

* Lain 的每一个 App 会新建一个 Calico Profile，保证 App 间互相隔离，同时对这个 Profile 初始化一些标准的 ACL 规则（由 Console 完成）
* 创建容器时，Console 会告知 Deploy 需要使用的网络，由 Deploy 组装容器 Spec 的网络部分
* 如果是升级或迁移操作，Deploy 会设置使用上一个 ip，否则使用自动分配
* Docker damon 收到 Create 请求后，调用 Calico 的 libnetwork 插件，分配 ip 并初始化网络
* 容器内访问容器外资源，使用 iptables 的 SNAT 完成，由 Calico 保证
* 容器外访问容器内资源，使用 iptables 的 DNAT 完成，由 networkd 保证


## 已知问题
1. 由于 Calico 的 libnetwork 插件也同时跑在了 Docker 上，如果 Docker 意外重启，会导致网络信息无法自动清理，同名容器只能删除重建

