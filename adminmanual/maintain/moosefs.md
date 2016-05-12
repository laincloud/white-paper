# 使用MooseFS


## 使用已有的MooseFS(推荐)

```
etcdctl set /lain/config/moosefs 192.168.77.21:9421
# 或 
etcdctl set /lain/config/moosefs 192.168.77.21 # 端口默认9421
```

## 创建MooseFS集群

我们可以在lain集群的节点上构建一个moosefs服务。但不建议这样做。

这里对MooseFS的支持仅限于初始化, 关于集群的维护等操作并不支持。

### 1. 先初始化etcd上的数据

```bash
# set the moosefs master
etcdctl set /lain/nodes/moosefs-master/node1:192.168.77.21:22 node1:192.168.77.21:22

# set the moosefs chunkserver
etcdctl set /lain/nodes/moosefs-chunkserver/node1:192.168.77.21:22 node1:192.168.77.21:22
etcdctl set /lain/nodes/moosefs-chunkserver/node2:192.168.77.22:22 node2:192.168.77.22:22

# set the moosefs metalogger
etcdctl set /lain/nodes/moosefs-metalogger/node2:192.168.77.22:22 node2:192.168.77.22:22
```

初始化后，该moosefs集群的master会自动配置`/lain/config/moosefs`, 无需再手动设置

### 2. 执行ansible

```sh
ansible-playbook -i playbooks/cluster -e "role=moosefs-build" playbooks/role.yaml
```

该命令会根据etcd中的配置，在指定的节点上启动moosefs server.


## 使用MooseFS

有了 MooseFS Service 后，使用以下命令,将`/mfs`挂载到MooseFS。 此后再通过`add-node`新增节点，新节点的`/mfs`也会被自动挂载上

```sh
ansible-playbook -i playbooks/cluster -e "role=moosefs" playbooks/role.yaml
```

## Register On MooseFS

bootstrap完后，Registry默认是使用local filesystem作为backend，这也导致它注定是个单点。

当Lain有了MooseFS后，可将Registry的backend改为MooseFS，此后我们就可以对Registry进行scale，实现高可用。

方法如下:


```bash
ansible-playbook -i playbooks/cluster -e "role=registry-moosefs" playbooks/role.yaml
```

**注1:** 该操作执行过程中，Registry是无法对外提供服务的。因为我们需要将已有的数据拷贝到MooseFS上，这需要花一些时间。

**注2:** 这个过程是不可逆的, Lain不支持自由灵活的改变Registry的backend。如果想把Registry再从MooseFS上搬走，请管理员自己设计搬迁方案。
