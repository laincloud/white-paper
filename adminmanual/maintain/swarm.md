# swarm管理

## swarm manager

Lain集群在初始化时，默认当前机器就是一个swarm manager，会用docker启动一个swarm manager.

同etcd集群管理类似，swarm可通过修改etcd中`/lain/nodes/swarm-managers`下的内容，来增删swarm manager。

### 查看所有manager
```bash
etcdctl ls /lain/nodes/swarm-managers
```
### 查看leader
多个swarm manager有一个leader，其他都是follower。真正工作的只有leader，其他follower会将请求代理到leader上。当leader故障后，剩下的follower会自动选举出一个leader。
```bash
etcdctl get /docker/swarm/leader
```
### 增删manager 
```bash
# 以 node2:192.168.77.22:22 为例子

# 增加一个manager
etcdctl set /lain/nodes/swarm-managers/node2:192.168.77.22:22 192.168.77.22
ansible-playbook -i /vagrant/playbooks/cluster -e role=swarm  /vagrant/playbooks/role.yaml

# 删除一个manager
etcdctl rm /lain/nodes/swarm-managers/node2:192.168.77.22:22
ansible-playbook -i /vagrant/playbooks/cluster -e role=swarm  /vagrant/playbooks/role.yaml
```

## swarm agent
todo
