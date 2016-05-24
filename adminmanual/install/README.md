# Production Install

## Prerequirements

Lain 理论上可以运行在任何 Linux 发行版之上，只需要满足如下条件：

1. 使用 systemd
1. Linux 内核版本 >= 3.10 （for [Docker](https://www.docker.com/))
1. 有 curl 命令行可用

例如 CentOS 7 即满足以上条件。

*注：因为时间的原因暂时只在 centos 7.1+ 上进行了测试，建议使用 centos 7.1 以上版本*

### 公有云支持

>未列举不代表不支持，只是还没有验证

- [aliyun](aliyun/)
- [qingcloud](qingcloud/)
- [aws](aws/)

## 生产环境的规划和配置

>大家既然看到这一章就是为了在生产环境中使用 LAIN
>那必然需要考虑到方方面面的事情
>下面就以 aliyun 为例讲述一下如何进行 LAIN 的生产环境配置
>并给出一个具有一定可用性/安全性/可行性的生产集群规划方案

### 1 规划 LAIN 集群的网络拓扑以及域名设定

>建议先阅读 [Domains And SSL](../domainandssl/) 一章了解 LAIN 集群里的几种类型的域名以及配置方法

*以 Aliyun 部署为例*

#### 1.1 举例：一种对可用性，安全性有所考虑的网络拓扑设计

![LAIN Aliyun ](img/LAIN-Aliyun-Topology.png)


#### 1.2 补充：上述网络拓扑设计的预设以及详细解说

*预设*

- 一个 startup 公司，假设叫做 PowerLain 公司
- 他们打算用 Aliyun 搭建 LAIN 集群来部署自己的互联网服务

*可用性考量*

- 最前端采用 SLB
- LAIN web 入口 HA
- LAIN master HA
- etc.

*安全考量*

- 为了安全因素考量，对用户服务的域名会单独购买，不使用 LAIN cluster domain 的子域名，也就是说:
    - LAIN cluster domain 生成的默认应用域名，希望只在公司内网（或者更干脆的只在 Aliyun 内网能访问）
    - 对用户服务的域名，都单独购买域名和 SSL 证书，通过 web Proc 的 mountpoint 功能和 LAIN 集群的 ssl 设置来响应外部服务
- 利用 LAIN 集群的 webrouter 组件的内建机制，物理分隔用户服务的外网域名和内网服务的内网域名之间的入口，避免暴露内部服务到外网
    - webrouter 内建机制，对 mountpoint 中指定的外网域名（判断标准是不在 `lain` `lain.local` 以及 `etcdctl get /lain/config/extra_domains` 结果列表内），建立 nginx 配置时额外响应 8080/8443 端口组，而内网域名只响应 80/443 端口组
    - Aliyun 建立外网 SLB
        - 配置 DNS 解析，对 mountpoint 中指定的外网域名解析到外网 SLB 公网 ip
        - dnat 外网访问外网 SLB 公网 ip 80/443 的流量到 webrouter node 的 8080/8443 端口，webrouter node 可按照 webrouter scale 的攻略横向扩展
    - Aliyun 建立内网 SLB
        - 配置 DNS 解析，对 LAIN 集群标准域名的统配域名解析到内网 SLB 入口 ip
        - dnat 外网访问内网 SLB 入口 ip 80/443 的流量到 webrouter node 的 80/443 端口，webrouter node 可按照 webrouter scale 的攻略横向扩展
- 在 Aliyun LAIN 集群的同一个 VPC 建立一个 lain-box 节点，安装好 LAIN CLI，用来进行 LAIN 集群上应用的管理

### 2 为 Docker 配置 devicemapper direct-lvm

关于 devicemapper 的详细内容可参考 docker [官方文档](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/)。

默认情况下，docker 底层使用的 devicemapper 以 loop-lvm 模式启动，这是官方**强烈不推荐**在生产环境使用的。所以，建议每个 lain-node 都为 docker 准备一个 100G+ 空间大小磁盘，以供为 devicemapper 配置 direct-lvm 使用。

*以 Aliyun 为例*

1. 为服务器实例创建新磁盘，建议 100G 以上。
2. bootstrap 时候设置 `--docker-device` 参数。
   假设我们添加的新磁盘设备为 `/dev/vdb`，则 bootstrap 命令为：
   ```bash
   ./bootstrap -r registry.aliyuncs.com/laincloud --docker-device=/dev/vdb --ipip
   ```
3. bootstrap 成功后，`docker info` 查看配置。如果不是 `direct-lvm` 信息中会有类似下面的WARNING:
   ```sh
   WARNING: Usage of loopback devices is strongly discouraged for production use. Either use `--storage-opt dm.thinpooldev` or use `--storage-opt dm.no_warn_on_loop_devices=true` to suppress this warning.
    ```

增加节点同理，指定 `--docker-device` 参数，新节点的 docker 就会使用 direct-lvm。如：
```bash
lainctl node add -p playbooks --docker-device=/dev/vdb node2:192.168.77.22
```
