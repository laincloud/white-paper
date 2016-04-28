# etcd集群管理

一个Lain集群有且只有一个etcd-cluster支持。每一个Lain集群中的node上都会运行一个etcd，或是etcd-member，或是[etcd-proxy](https://github.com/coreos/etcd/blob/master/Documentation/proxy.md)。

所以在任何一个node上，只需通过 `http://localhost:4001` 即可访问etcd-cluster

## 初始化

Lain集群初始化时，会启动一个etcd节点，此时是一个单节点的etcd-cluster。

此后每一个新加入Lain集群的node上，都会在加入时启动一个etcd，以proxy身份连入etcd-cluster

## 查看member

```bash
etcdctl member list
```
或
```bash
etcdctl ls /lain/nodes/etcd-members  # 结果现实节点为member
```

## resize etcd cluster
如上所说，etcd cluster只有一个member，其他的node上全都是proxy。此时的etcd cluster并不是高可用的。

Lain支持手动修改各个node上etcd的mode，即把一个etcd-proxy变成etcd-member，以提高可用性；或将etcd-member改成etcd-proxy以提高性能。

以`node2:192.168.77.22:22`为例(可通过`etcdctl ls /lain/nodes/nodes` 查看所有节点)。

### proxy 变 member
```bash
etcdctl set /lain/nodes/etcd-members/node2:192.168.77.22:22 192.168.77.22
ansible-playbook -i /vagrant/playbooks/cluster -e role=etcd  /vagrant/playbooks/role.yaml
```

### member 变 proxy

```bash
etcdctl rm /lain/nodes/etcd-members/node2:192.168.77.22:22
ansible-playbook -i /vagrant/playbooks/cluster -e role=etcd  /vagrant/playbooks/role.yaml
```
