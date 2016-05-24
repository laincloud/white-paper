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

*沙盘演练*

- Preconditions:
    - PowerLain 公司打算做一个公司首页，域名和 SSL 证书都已买好
        - powerlain.com
        - powerlain.com.key && powerlain.com.crt
    - PowerLain 公司已有 Aliyun 账户
        - 先创建了一个 VPC ，配套的网络和路由器也创建好，选定网段是 192.168.77.0/24
        - 创建一个安全策略，为了避免麻烦粗放的定为放行所有流量（仅为方便，实际生产中应该尽可能多做隔离）
        - 先弄一个 LAIN 节点的模板镜像
            - 在上述 VPC 里创建一个 ECS ，使用 centos 7 模板，按需收费，根分区 20 G （各项都仅供参考）
            - yum update
            - yum install -y epel-release
            - 等都 OK 之后保存为一个自定义的模板镜像，姑且命名为 lain-node-tmpl
        - 创建一个外网 SLB （负载均衡），假设其公网 ip 为 ex-SLB-ip
        - 创建一个内网 SLB （负载均衡），假设其入口 ip 为 in-SLB-ip
    - PowerLain 公司打算在 VPC 里面搭建 LAIN 生产集群
        - 直接使用 LAIN 的默认的集群标准域名 `lain.local` 进行集群控制和应用部署，采用内部 DNS 劫持或者直接写 `/etc/hosts` 的方案进行解析
        - 在 LAIN 集群所在的 VPC 另外创建一个管理跳板机，上面部署 openvpn ，以此让办公网可以通过 vpn 连接生产网络

- Steps:
    - 搭建 LAIN 集群
        - VPC 里使用 lain-node-tmpl 模板创建 ECS node：lain-01 ，资源需要 2U4G 以上，`/` 分区 20G （此配置仅用于演示）
            - 正式的生产节点配置见 [生产节点配置](productionnode/)
        - `@lain-01` 改好 hostname 等 `hostname -s lain-01` 之后 relogin
        - 假设 lain-01 的 node ip 是 192.168.77.21
        - `@lain-01` `cd lain` 然后 `./bootstrap -r registry.aliyuncs.com/laincloud --ipip`
            - 如果使用正式的 [生产节点配置](/productionnode) ，则使用 `./bootstrap -r registry.aliyuncs.com/laincloud --ipip --docker-device=BLAHBLAH` 来指定 lvm pv 给 docker daemon ，具体见生产节点配置的 REF
        - 同样的配置创建 lain-02 lain-03 ，并加入集群，假设他们的 ip 是 192.168.77.22 192.168.77.23
            - `@lain-01`: lainctl node add -p playbooks -q lain-02:192.168.77.22  # 正式生产也许也用上 `--docker-device` 选项
            - `@lain-01`: lainctl node add -p playbooks -q lain-03:192.168.77.23  # 正式生产也许也用上 `--docker-device` 选项
    - 搭建 管理跳板机 lain-baseton
        - 使用 lain-node-tmpl 模板在 VPC 里创建跳板机
            - 参考 [lain-box 的构建方式](https://github.com/laincloud/lain-box/tree/master/builder) 安装各种依赖等
            - `@lain-baseton` 执行 `lain config save local domain lain.local` 和 `lain config save-global private_docker_registry registry.lain.local`
    - 设定内网 SLB
        - 将 webrouter 所在的节点作为其后端，in-SLB-ip 的 80/443 端口对应 webrouter 所在节点的 80/443 端口
        - 内网设定 *.lain.local 的 DNS 劫持到 in-SLB-ip
    - 可选：在 `lain-baseton` 上 [webrouter scale](maintain/webrouter/)
        - 将新增的 webrouter 所在节点加入到内网 SLB 后端中
    - 设定公网 SLB
        - 将 webrouter 所在的几个节点作为其后端，ex-SLB-ip 的 80/443 端口对应 webrouter 所在节点的 8080/8443 端口
        - 设定 powerlain.com 的 DNS 解析到 ex-SLB-ip
    - 可选：打开 LAIN 集群的 auth 机制，参见 [auth](../auth/) 和 [sso](../sso/)
        - 打开 LAIN 集群的 auth 机制之后 lain-baseton 上进行操作时也需要进行 lain login 等操作
    - 在 `lain-baseton` 上进行 powerlain.com 网站的开发，通过 LAIN CLI 部署到 LAIN 集群
        - `lain.yaml` 的内容 DEMO
            ```
            appname: powerlain
            build:
                base: incloud/centos-lain:20160503  # 这个 base image 里包含了 golang/python/nodejs 环境
                script:
                    - go build -o powerlain
            web:
                port: 8000
                memory: 256m
                num_instances: 1
                cmd: /lain/app/entry.sh
                env:
                    - COPYRIGHTYEAR=2016
                mountpoint:
                    - powerlain.com
            ```
        - 具体的构建和发布过程请参考
            - [app-demo](../../quickstart/app-demo/)
            - [IntoTheLAINStepbyStep](../../quickstart/stepbystep/)
            - [LAIN Tour](../../usermanual/tour/)
            - [SDK && CLI](../../usermanual/sdkandcli/)
        - `lain ps local` 即可查看部署结果
        - 此时应可通过 `powerlain.lain.local` (前提是进行了 DNS 劫持，或者写了 `/etc/hosts`) 或者 `powerlain.com` 对部署好的网站进行访问
    - 可选：本地安装 `lain-box` ，并通过自己在 `lain-baseton` 上搭建 `openvpn` 的方式连入到 VPC 内网，处理好 DNS 解析之后即可在本地进行 LAIN 的应用开发管理
