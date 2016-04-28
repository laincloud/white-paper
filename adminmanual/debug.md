# Lain 组件 debug 说明

> # FIXME

### Layer 0

#### docker daemon(孙毅, 陈云飞)

> # FIXME

**docker daemon挂了：**

- 重启, 手动
    - `docker start swarm-agent.service calico-node rebellion`
    - `docker -H :2377 start xxxx` 启动其他的业务container
    - 启动后 `docker exec <containerid> ip addr` 查看是否分配了ip, 如果没有就停掉重启

#### etcd(陈云飞)

- **链接不上etcd:**  `systemctl start etcd`
- **etcd服务起不来:** 说明它是个member， proxy一般不会出什么问题。 可把这个member从cluster中remove，然后再add回去。步骤如下

    1. `etcdctl cluster-health`会发现一个`unhealth`的节点
    2. `etcdctl member remove <id>`, id为上一步结果中的第二列，形如`4f0fdd215b8b821d`
    3. 删掉异常节点的etcd数据, 已`node1`为例, `rm -rf /var/etcd/node1`
    4. 把节点重新加回集群，还是以`node1`为例， `etcdctl member add node1 http://192.168.77.21:9001`(node1和地址换成指定的)
    5. `etcdctl cluster-health`检查集群是否正常

- **看member之间的关系:** `curl http://localhost:4001/v2/stats/self`或`curl http://localhost:4001/v2/stats/leader`

#### moosefs(陈云飞, flex, 孙毅)

- **/mfs没有挂载** `mfsmount /mfs`*（每台机器都要检查）*
- **把registry的data挂到moosefs上**: `mount -B /mfs/lain/registry /var/lib/registry`, 使用`mfsdirinfo /var/lib/registry`检测是否挂到moosefs上.*(每台机器都要检查)*

#### lainlet(陈云飞)

- **感觉拿到的数据不太对(和etcd不一致):** 重启lainlet, `systemctl restart lainlet`

#### swarm(陈云飞, 孙毅)

- **swarm链接失败:** 重启,`systemctl restart swarm-manager.service`
- **swarm info发现少node: **`etcdctl ls /lain/swarm/docker/swarm/nodes`查看是否真的缺少node
    - 是，去丢失的机器上重启swarm-agent
    - 否，重启swarm-manager

#### networkd(徐涛)

> # FIXME

```
ansible -i cluster all -m shell -f1 -a 'iptables -t nat -n -L lain-OUTPUT &>/dev/null || iptables -t nat -N lain-OUTPUT'
ansible -i cluster all -m shell -f1 -a 'iptables -t nat -n -L lain-PREROUTING &>/dev/null || iptables -t nat -N lain-PREROUTING'
ansible -i cluster all -m shell -f1 -a 'iptables -t nat -C PREROUTING -j lain-PREROUTING &>/dev/null || iptables -t nat -A PREROUTING -j lain-PREROUTING'
ansible -i cluster all -m shell -f1 -a 'iptables -t nat -C OUTPUT -j lain-OUTPUT &>/dev/null || iptables -t nat -A OUTPUT -j lain-OUTPUT'
ansible -i cluster all -m shell -f1 -a 'systemctl restart networkd'
```

#### rebellion(白渐)

目前rebellion容易出问题的地方是新部署（或更新）的应用的logs配置没有正确地被刷到rebellion.service的container中的/etc/flume.cnf文件中

此时应按如下步骤定位问题：

- 在container中按如下方式检查lainlet中是否能获得kafka地址
    - `curl -XGET http://lainlet.lain:9001/v2/configwatcher?target=kafka`如果没有任何结果，说明etcd中未配置kafka地址，请联系徐卓夫同学配置。etcd的kafka配置项是/lain/config/kafka
    - 在container中按如下方式检查lain let中能否获得proc的logs配置信息`curl -XGET http://lainlet.lain:9001/v2/coreinfowatcher`配置信息在spec的annotation字段中，如果没有说明该应用信息未被完整写入etcd
    - 如果以上步骤均通过，说明是rebellion更新程序本身的bug，由于rebellion是无状态的，所以可以直接通过systemctl重启。`systemctl restart rebellion`

### Layer 1

#### console(李盼, 孙毅)

> # FIXME

能够进入 `http://console.domain/archon/` 查看 app 运行状态说明 console 运行正常

可能出现的问题是：

- **console 页面不能显示**：dns 解析出问题，可能是 networkd 或者 webrouter 出现状态，进集群查看 iptables 规则
- **console 页面不能展示数据**：deploy 出现问题，可尝试重启 deploy
- **console 注册 app 显示 `sso server error` **：说明 sso 中已存在相应 group，但是 console 中并未纪录存在此 app，出现了数据不一致情况： 一般是测试时候 开启了 auth 容易出现此问题，确认无影响后可以将 sso 中对应 group 手动删除

#### deploy(米嘉, 陈云飞, 孙毅)

> # FIXME

**deployd没有把确实的container拉起来，可能是:**

- deployd container挂了，可尝试重启，并查看log`tail -f /var/log/message | grep <id>`
- swarm-manager的问题，检查swarm-manager是否能链接上，`docker -H :2376 info`看资源够不够等信息
- webrouter挂了
- registry挂了
- **deployd的container没了**

#### registry(李盼, 孙毅)

可以使用 `curl http://registry.domain/v2/` 查看 registry 能否访问

在关闭 registry auth 的情况下，可以使用 `curl http://regsitry.domain/v2/registry/tags/list/` 查看能否 registry 数据读取是否正常

可能出现的问题：

**registry 不能往上 push** ，则可能原因：

- 开启 auth 的区域首先确认用户名密码正确
- 可能 mfs 被 unmount 掉了，去宿主机上 `/var/lib/registry` 查看是否存在
- 磁盘空间被占满，进入宿主机运行 `docker info` 查看磁盘空间，如果磁盘空间有问题，需要进行磁盘清理

**输入用户名、密码正确，但 registry 一直报 unauthorized**，可能原因：

- 一般是 prod 区可能出现此问题，一般是由于 sso 不能访问 prod 区的 console，首先确认 sso 所在宿主机能够访问到 console，不能访问则需要手动设置 hosts
- 如果设置 hosts 不能解决问题，可以先将 registry auth 关闭，等调整后再进行恢复，关闭方法 TBD

#### tinydns(白渐)

当同一个App内的container通过ping proc_name-instance_no时出现unknown host错误，此时基本可以确定是tinydns出现了问题。需要进入tinydns的container查找问题根源。

- 先cat /etc/ndjbdns/data，看一下是否有之前无法ping通的proc instance的相关配置，且相关配置中是否有IP地址，IP地址是否和该container的IP相同？
- 如果没有IP地址或IP地址和container的实际IP不一致，说明此时etc出现了脏数据，这时候应该在console中通过伪upgrade的方式(例如scale up 1MB)让deployed重建该container解决。
- 如果没有该proc instance的配置，有可能是dns-creater出现问题卡住了，可以在container中通过重启该程序解决。`supervisorctl restart lain-dns`

#### lvault(王超一)

lvault 状态查询：

首先，查看 http://lvault.{LAIN_DOMAIN}，看 UI 能否显示接着可以点击自助服务的 LVAULT 集群状态查询，要保证至少有一个 vault node 的未解锁是 false，至少有一个 lvault 的不可用是 false. 如果不满足这两个条件之一，请用 unseal.sh 解锁.

lvault 分为两个应用，lvault 和 vault. 已经在 lvault 中实现了一些简单的 vault 集群维护的 api.根据经验，lvault 出问题的情形主要是网络问题，大部分情况可以看标准输出的 log 解决。

但是，当集群所有 container 重启时，比如断电，问题会比较严重，这时 lvault 丢掉了内存中的 token 和 key, 需要管理员操作，已经写了个脚本 unseal.sh, 以开发区为例，用法是 ./unseal.sh lain.bdp.cc.具体 TBD

#### webrouter(白渐，孙毅)

> # FIXME

>`webrouter` 不正常的话集群的应用基本就不可访问了，是最高优先的问题

- 首先确认 webrouter 在哪个节点上 `docker -H swam.lain:2376 ps -a |grep webrouter`
- 看看 webrouter 有没有分到 calico ip `docker exec ${webrouter_container} ip addr`
- 如果有没有 calico ip 那说明可能是挂掉了被 deployd 拉起来后没分到 ip ，那就重新启动一下看看
    - `docker -H :2377 stop ${webrouter_container}`
    - `docker -H :2377 start ${webrouter_container}`
- ip 没问题就看看是不是配置生成的有问题了
    - `docker-enter ${webrouter_container}`
    - `supervisorctl status` 看看 tengine / watcher 的状态
    - `tengine -t` 看看自动生成配置文件是不是有问题，有的话看看是什么问题
    - 配置没问题那么尝试 `supervisotctl restart watcher` 重启一下 watcher ，如果是 watcher 卡住了，那么这样应该可以解决
- 如果 webrouter 的容器消失了那么就要 check 各个配置
    - `etcd` 配置 ( `etcdctl get /lain/deployd/pod_groups/webrouter/webrouter.worker.worker` ) 来看 deployd 为什么没把 webrouter 拉起来（如果出现这样的情况，很可能是 etcd 或者 deployd 的问题 ）
    - `docker logs -f swarm-manager.service` 看看 swarm 的日志是不是有异常


### Layer 2

#### MySQL-Service（白渐）

MySQL-Service由三大部分组成: monitor负责监控各个Server状态，portal负责代理外部app请求，server是实际工作的mysql instance。

##### monitor故障

故障表现：访问http://mysql-service.LAIN_DOMAIN (首次进入需要登录SSO)，观察overview里面mysql-server的状态是否都是OK

如果均是ERROR，则有可能是lainlet无法获取mysql-server的proc信息或mysql-service.web.web无法ping通mysql-server-*INSTANCE_NO*

故障定位:

- 在mysql-service.web.web中`curl -XGET http://lainlet.lain:9001/procwatcher/?appname=mysql-service`如果没有得到mysql-server的proc信息则说明lainlet 有问题
- 在mysql-service.web.web中`ping mysql-server-1`如果出现unknown host，说明是tiny-dns出现问题

##### mysql-server故障

- 一般情况下mysql-server由于只有一个mysql进程，因此故障基本上都是OOM。可以`docker ps`看一下mysql instance最近是否有被重启的痕迹，然后`dmesg -T`看一下最近是否发生了mysql的OOM。如果是OOM就需要在console中scale up内存
- 如果mysql-server在迁移后集群出现了故障，恢复时一定要确保所有的数据库都存在于/var/lib/mysql/*$database_name*中，如果没有说明数据迁移不完整，需要重新进行数据迁移。此时先去原节点确认上面的数据是否完整，如果完整则先断开新mysql集群的同步，然后删除新集群的所有数据文件，最后停掉所有的mysql instance并将原数据rsync到新集群的volume目录下。如果原节点的数据不完整，则只能通过数据恢复流程从之前的数据备份恢复

##### proxy故障

- 在use_service app的container中```ping mysql-master```,如果出现unknown host错误说明是tiny-dns出现了问题
- 如果能ping通但是连不上mysql，需要去该app的portal中看/var/log/proxy.INFO，如果最近连续出现类似于**Waiting for targets. Reconnect to monitor in 10s**的日志，说明是mysql-service的monitor出现了问题。

