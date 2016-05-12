# 管理员操作

现阶段大多数 Lain 管理员操作需要使用相应 ansible 脚本进行处理，后续会提供管理员命令行工具进行处理

## DNS 特定域名劫持

集群中存在某些需要进行特定劫持的域名地址，可以通过以下方式进行劫持：

首先在etcd中进行设置，如：

```bash
etcdctl set /lain/config/dnshijack/<domain.com> '<ip>'
```

有多个需要劫持的地址则设则多次，设置完之后可以通过运行如下脚本会将相应配置放于`/etc/dnsmasq.d/proxy.conf`中：

```bash
ansible-playbook -i playbooks/cluster -e "role=dns-hijack" playbooks/role.yaml
```

**注意:** 每次运行该脚本会将原有配置中内容覆盖掉，因此如果发生变动请务必记得修改etcd中的值，不要直接修改配置文件


## swarm.lain 域名劫持

目前 Lain 中 swarm manager 使用的是伪HA策略，当 swarm 集群 出现主节点更换的情况下，需要管理员手动切换 swarm.lain 对应的 ip，可以使用如下命令：

```bash
ansible-playbook -i playbooks/cluster -e "role=swarm-hijack" playbooks/role.yaml
```

此命令会将 etcd 中 `/lain/config/swarm_manager_ip` 的 value 刷成 swarm.lain 对应的ip


## lain 节点清理

lain 节点清理脚本可以将 `/var/log/messages` 多余的备份文件清除，以及清除各节点冗余的 app image。

Image清理过程会首先查看该节点目前运行了哪些app，然后会将该app在该节点上保留最多3个版本的image，其余的都将被删除。可以使用如下脚本清理node1节点：

```bash
sudo ./clean-node node1:192.168.77.21
```

clean-node 脚本支持同时清理多个节点， 使用 `--all` 参数可以同时清理所有节点；
**注意:** 在脚本运行过程中会需要管理员交互的地方，需要管理员check存在的app


## syslog 日志文件位置迁移

目前syslog 产生的日志文件（默认是`/var/log/messages`）较大，可能会导致对应目录容量吃紧，此脚本可以将 rsyslog 的日志文件存储位置到指定的其它地方去：

```bash
ansible-playbook -i playbooks/cluster -e "role=rsyslog-relocate" -e "syslog_messages_location=/data/log/messages" playbooks/role.yaml
```

**注意:** `syslog_messages_location`为迁移后日志文件的地址，不设置的话，地址不变


## 重启节点后状态恢复

如果 lain 某个节点出现了重启的情况（如断电了），需要有机制将 node 重新变成可调度状态。可执行以下命令：

```bash
ansible-playbook -i /vagrant/playbooks/cluster /vagrant/playbooks/site.yaml
```

脚本运行完毕后可以通过运行 `docker -H :2376 ps | grep calico` 监测对应节点是否启动

如果 deployd 同样处在该节点且 deployd 不可用，则需使用`systemctl start deployd` 拉起容器，此后即等待该节点相关容器被 deploy 拉起即可。

如果脚本不能将集群状态恢复，可以进行手动修复，参考[系统恢复](recovery.html)

**注意:** 如果该节点包含 registry ，则 ansible 脚本不能成功运行，需要参考[系统恢复](recovery.html)进行手动修复，或者在将 registry 拉起之后再运行脚本
