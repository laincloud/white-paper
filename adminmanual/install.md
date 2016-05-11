# Production Install

### Prerequirements

Lain 可以运行在任何 Linux 发行版之上，只需要满足如下条件：

1. 使用 systemd
1. Linux 内核版本 >= 3.10 （for [Docker](https://www.docker.com/))
1. 有 curl 命令行可用

例如 CentOS 7 即满足以上条件。

建议使用 centos 7.1 以上版本

### 安装第一个节点

把 lain 的源码 clone 到节点的某个目录下，用 root 运行

```sh
./bootstrap --node-ip 10.100.143.67 --virtual-ips=10.100.143.201,10.100.143.202,10.100.143.203 --domain lain.example.com --net-interface eth0
```

`--domain` 为应用部署后默认的域名。如果不指定，默认为 lain.local。Lain 系统会提供该域名及其下子域名的解析服务。此域名可以是私有域名，不需要公网可解析。

`--vip` 为集群对外提供服务的虚 IP。在本例中，负责解析 lain.local 的 DNS 需要将 `*.lain.local` 的 A 记录查询结果均设置为此 IP，这样才能从外部访问 lain 上部署的服务。

该脚本将执行以下内容：

1. 安装必要的依赖，构建打包 lain;
1. 安装和配置 lain 的运行时依赖;
1. 生成 lain 集群 id，创建
1. 安装和配置 lain 。
1. 部署基础的 lain apps，比如 docker registry、nginx、管理服务等。

在bootstrap结束后，部署其他app时可以选择是否将sso验证服务打开，在集群中通过设置相应的etcd值能够达到此目的
```
# 打开验证服务，此时需要在下面所有需要认证服务的操作的curl命令后加入 '-H "access-token: ********"'
etcdctl set /lain/config/auth '{"type": "lain-sso", "url": "http://sso.example.com"}'
# 关闭验证服务，此时，直接按后续步骤即可完成操作
etcdctl rm /lain/config/auth 
```
access-token的值是从sso系统中得到的
"url"中的值可以被其他提供sso验证的url替代

此时即已经有一个可用的 lain 系统了 :-)


### 外部访问

*此处所说的外部访问是指 lain 集群节点之外的访问，比如从企业内部其他服务器或者办公电脑*

虽然此时在 lain 节点上已经可以解析 lain.example.com 的子域名，但企业内部其他计算机尚不能解析（比如开发者的桌面电脑），需要在企业内网 DNS （即负责解析上级域名 example.com 的域名服务器）上配置通配符解析 \*.lain.example.com A 记录到集群虚 IP 上（此例中为 10.1.2.3）即可。

### 公网访问

如果需要公网能够访问 lain 上的应用，首先需要应用本身在 lain.yaml 中配置绑定公网域名(TODO 连接 mountpoint 文档)。

如果在 lain 系统外已经有公网可见的负载均衡器或者反向代理，可以直接配置使用上述虚 IP。

如果没有，而 lain 节点本身是有公网接入的，则可以增加一个公网的 VIP(TODO 连接 VIP 文档)，并让公网DNS解析到此VIP上即可。


### 增加节点

比如要将 192.168.77.22 这台服务器加入集群，需要先把 lain 的管理账号的 SSH 公钥复制到 root 账号下，以允许在管理节点（即执行 bootstrap 的节点）上用 lain 账号管理。建议此步可以作为系统安装的一部分，如果尚未复制公钥，可在管理节点上执行如下命令来复制:

```sh
sudo ssh-copy-id -i /root/.ssh/lain.pub root@192.168.77.22
```

在管理节点运行:

```sh
lainctl node add -p /path/to/playbooks node2:192.168.77.22
```

即可将 192.168.77.22 加入到 lain 集群中，命名为 node2 。


## 删除节点

系统做以下事情：

1. 确保该节点不是正在工作的 etcd memeber；
2. 确保该节点不是正在工作的 swarm manager；
3. 从集群中将该节点摘除。

在管理节点运行:

```sh
lainctl node remove -p /path/to/playbooks node2
```

即可移除 node2 这个节点。
