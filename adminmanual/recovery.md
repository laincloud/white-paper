# Lain 集群修复

此文档为 Lain 管理员修复集群状态参考手册，主要展示如下问题:

- Lain 集群集中常见错误及处理方式
- 如何确认 Lain 集群中各组件运行状态
- 组件出现状态可能会出现哪些问题
- 如何修复组件以快速保持 Lain 集群稳定

## 集群常见问题

### 磁盘满了

>参考 Lain 节点清理

### 某个节点挂掉

>参考 节点恢复

## 集群停电重启后状态组件恢复顺序

### 确认系统时间

- `date`
- 如果时间不正确则需要 `date -s '2015-11-30 10:29:00'` 类似这样设置

### 确认节点 dns 设置

>/etc/resolv.conf

- `modprobe ip6_tables`  # 确保内核模块加载
- `head -n1 /etc/resolv.conf |grep 127.0.0.1`  # 确认文件第一行存在 127.0.0.1 ，不存在就编辑进去

### iptables

- `iptables -t nat -n -L` 查看iptables规则是否有异样，可与正常node上的结果进行下对比。
- 若INPUT chain 有问题， 执行`ansible-playbook -i playbooks/cluster -e "role=firewall" playbooks/role.yaml` 对INPUT chain 进行重新初始化。

### etcd 集群

- master/member node
    - `etcdctl member list`
    - `etcdctl cluster-health`
- proxy node 
    - 确认是 proxy 节点
        - `systemctl stop etcd`
        - `rm -rf /var/etcd/$(hostname)`
        - `systemctl restart etcd`
- member node
    - 常规来说直接能起
    - 不能起，比如数据 crash 了，则需要需要判断其他member是否正常，然后进行 etcd member 自己的修复过程，把这个member从cluster中remove，然后再add回去。步骤如下:
        - 在正常的 etcd member 上： `etcdctl cluster-health`会发现一个`unhealth`的节点
        - 在坏掉的 etcd member 上： systemctl stop etcd
        - 去一个正常的 etcd member 上： `etcdctl member remove <id>`, id为上面结果中 unhealth 的第二列，形如`4f0fdd215b8b821d`
        - 回到坏掉的 etcd member 上： rm -rf /var/etcd/$(hostname) ，以 `node1`为例, `rm -rf /var/etcd/node1`
        - 正常的 etcd member 上：  把节点重新加回集群，还是以`node1`为例， `etcdctl member add node1 http://192.168.77.21:9001`(node1和地址换成匹配的)。
        - 在那个坏掉的 etcd member 上： systemctl start etcd
        - `etcdctl cluster-health` 检查集群是否正常。

### mfs 确保以下 service 状态

- mfsmaster- `cat /etc/hosts|grep mfsmaster`
    - `systemctl status mfsmaster.service`
- mfschunkserver
    - `etcdctl ls /lain/nodes/moosefs-chunkserver/`
    - `systemctl status mfschunkserver.service`
    - `ansible -i ~/lain/playbooks/cluster all -m shell -a 'systemctl status mfschunkserver.service'`
- mfs mount for registry
    - `ansible -i ~/lain/playbooks/cluster all -m shell -a 'mfsdirinfo /var/lib/registry'`
    - `ansible -i ~/lain/playbooks/cluster all -m shell -a 'mount -a'`
    - `ansible -i ~/lain/playbooks/cluster all -m shell -a 'mount -B /mfs/lain/registry /var/lib/registry'`

### lainlet

- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'systemctl status lainlet'`

### docker

- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'systemctl status docker'`
- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'systemctl start docker'`

### calico

> # FIXME

- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'docker ps |grep calico'`
- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'docker start calico-node'`
- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'docker -H :2377 ps |grep calico'`
- 出现问题可以手动删除容器之后重新部署
- docker -H :2377 ps 查看状态

### swarm

- agent
    - `ansible -i ~/lain/playbooks/cluster all -m shell -a 'docker ps |grep swarm'`
    - `ansible -i ~/lain/playbooks/cluster all -m shell -a 'systemctl start swarm-agent'`
- manager
    - `ansible -i ~/lain/playbooks/cluster all -m shell -a 'docker ps -a|grep swarm-manager'`
    - `systemctl start swarm-manager`
    - 查看状态 `docker -H :2376 info` 确认所有节点都存在

### networkd

- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'systemctl restart networkd'`
- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'systemctl status networkd'`

### tinydns- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'docker ps -a|grep tinydns'`- `docker -H :2377 start tinydns.worker.worker`- `etcdctl get /lain/config/vips/10.106.170.203` (需要一个查询 app vip 的工具?)- VIP `ip addr show dev eth0`

1. deployd- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'docker ps -a|grep deploy'`- `docker -H :2377 start deploy.web.web.v0-i1-d0`- `docker exec deploy.web.web.v0-i1-d0 ip addr`- `etcdctl get /lain/deployd/leader`- 等待 deployd 拉起剩余组件(需要查看 deploy 的工具?)

1. lvault 解锁- 运行两次 unseal.sh 脚本- `sh ~/unseal.sh yxapp.xyz`

1. 检查git/wiki是否运行正常- http://gitlab.yxapp.xyz- http://wiki.yxapp.xyz

1. 恢复 rebellion- `ansible -i ~/lain/playbooks/cluster all -m shell -a 'docker start rebellion.service'`

