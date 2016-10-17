# LAINCTL

集群的常规管理操作，如节点的添加与删除等。都可用 `lainctl` 完成。

lainctl 需要运行在 lain master 节点上，因为它要可能需要使用ansible playbook。


## 安装

lain 集群在 bootstrap 后时会自动安装 lainctl。手动安装或更新的方式:

```sh
git clone --depth=1 https://github.com/laincloud/lainctl.git
cd lainctl
python setup.py install
```


## 命令手册

### node

集群的节点管理。

- **list**

  `lainctl node list`

  列出集群的所有节点。

- **inspect**

  `lainctl node inspect NODE_NAME`

  查看一个集群节点的详细信息

- **add**

  `lainctl node add -p PLAYBOOKS [-P SSH_PORT] [-d DOCKER_DEVICE] [-c CID] [-s SECRET] [-r REDIRECT_URI] [-u SSO_URL] [-q] NDOE_NAME:NODE_IP [NODE_NAME:NODE_IP ...]`

  - `-p/--playbooks`: ansible playbooks 目录路径
  - `-d/--docker-device`: 初始化 `devicemapper` 的磁盘。若不指定，docker 使用 loopback device 作为存储启动。
  - `-q/--quiet`: 安静模式，不需要登录。
  - `-c/--cid`: 参考 [sso 手册](../usermanual/sso.html#应用注册)， 下同。
  - `-s/--secret`:
  - `-r/--redirect-uri`:
  - `-u/--sso-url`:

  为集群增加一个节点，假设新增 node 节点的 IP 地址是 `192.168.77.22`，我们将其命名为 node2。则增加节点的命令为:

  ```sh
  lainctl node add node2:192.168.77.22
  ```

- **remove**

  `lainctl node remove [-t TARGET] -p PLAYBOOKS NODE_NAME`

  - `-t/--target`: 如果节点上有swarm-manager，则 `--target` 需要被指定。
  - `-p/--playbooks`: ansible playbooks 目录路径
  - `NODE_NAME`: 节点名

  将一个节点从集群中移除。需要确保要移除的 node 上没有业务的 container，否则操作会被拒绝。业务的 container 可通过 `lainctl drift` 命令将其移到其他节点上。

  若被删除节点是一个 `swarm-manager`，需要对其进行迁移，`--target` 指定迁移的目标节点。

- **clean**

  `lainctl node clean -p PLAYBOOKS nodes [nodes ...]`

  - `-p/--playbooks`: ansible playbooks 目录路径

  清理一个节点上没用的image，释放磁盘空间。

  每个节点上会为业务保留最新3个版本的image，更老的image会被清空。以及已经不在这太机器上运行的业务的 image 也会被清空。


- **health**

  `lainctl node cluster`

  检查节点上各个基础组件运行是否正常。

### cluster

  `lainctl cluster health`

  检查集群的基础组件是否正常，包括 `etcd`, `swarm`, `deployd`, `console`。

### network

  `lainctl network recover -p PLAYBOOKS -n NODE -t TARGET_APP -P PROC_NAME [-i INSTANCE_NO] [-c CLIENT_APP]`

  - `-p/--playbooks`: ansible playbooks 目录路径
  - `-t/--target_app`: 要恢复的 appname
  - `-P/--proc_name`: 要恢复的 app 的 proc name，如果是 portal 类型的 proc，名字一般为 portal-{service_name}
  - `-i/--instance_number`: 要恢复的 proc 的 instance number，要恢复的 proc 不是 portal 类型时必须设置
  - `-c/--clint_app`: 要恢复的 service app 的 client appname，当需要恢复的 proc 是 portal 类型时必须设置

  当集群节点意外宕机时，可能会出现 docker 退出时没有回收相应容器的 endpoint 和 IP 的情况，当 Deployd 试图在其他正常节点恢复这些容器时，swarm 中会出现类似 `endpoint already exists` 的错误信息，这时可以使用此命令将 endpoint 及 IP 信息进行恢复。

### drift

  `lainctl drift [--with-volume] [--ignore-volume] -p PLAYBOOKS [-t TARGET] CONTAINER [CONTAINER ...]`

  - `-p/--playbooks`: ansible playbooks 目录路径
  - `--with-volume`: 是否迁移 volume 数据。如果设置，则在迁移 container 之前会把数据 rsync 到指定节点。
  - `--ignore-volume`: 是否忽略 volume 数据。如果设置，则 container 的 volume 会直接被忽略掉，直接迁移 container。
  - `-t/--target`: 要迁移到的节点。如果不设置，container 会随机迁移到一个节点上。当 container 存在 volume 且需要迁移，则 `--target` 不能被省略。
  - `CONTAINER`: container ID 或 名字


  把一个 container 迁移到另外一个节点上。

  在 container 有 volume 的情况下，`--with-volume` 和 `--ignore-volume` 有且只有一个可以被设置。

### auth

  `lainctl auth {open,close}`

  - `open`: 打开 auth
  - `close`: 关闭 auth

  注意：如果集群用了 'extra_domain', 并且 lain.local 在集群外不可访问时，打开 registry auth 时要指定其 realm 参数，如默认的 'http://console.lain.local/api/v1/authorize/registry/' 改为 'http://console.lain.cloud/api/v1/authorize/registry/'


  集群认证功能的开关设置。

### version

  `lainctl version`

  显示 lainctl 版本信息


 ### registry
 
 - **list**
 
   `lainctl registry list [-t target]`
 
   列出registry中所有的repo
   
   -t target 列出指定repo的所有tags
   
 - **clean**
 
   `lainctl registry clean [-t target] [-n num]`
 
   对registry 本地仓库进行清理，删除不用的images
   - `-t --target` 指定要被清理的repo(默认 all)
   - `-n --num`    指定repo需要保留的个数(默认 10)
   
   registry清理的基本操作：
 	1. 在主节点执行lainctl registry clean 标记要删除的镜像(保留最新的n份tag)
 	2. 将registry暂停服务或者设为read-only，然后在registry节点执行：registry garbage-collect config.yaml 进行文件实际删除
 	
    注意： 
  
     - 开了auth的registry，需要将主node节点ip 加入白名单:etcdctl set '/lain/config/registry_ip_whitelist' '${node1ip}'
