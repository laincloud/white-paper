# lain.yaml

>本节介绍 lain.yaml 如何编写以及 lain-cli 在编译 lain.yaml 时的 image 产物

## 1. lain.yaml 可选内容

在 LAIN 中创建一个 app 时，最重要的就是在应用的根目录下定义好一个正确的 lain.yaml。一个完整的 lain.yaml 可以包含以下内容：

```yaml
appname: {APP_NAME}             # 全局唯一的应用名
giturl: {GIT_URL}               # 将app绑定到一个具体的远程giturl，防止appname命名冲突，代码版本比较

build:                          # 描述如何构建应用 build image 
  base: {BASE_IMAGE}            # 一个已存在的 docker image ，包含编译环境和默认的配置     
  prepare:                      # 描述如何构建应用 prepare image
    script:                       # 定义构建 prepare image 时需要的 script
        - {PREPARE_SCRIPT}
    keep:
        - {RELATIVE_PATH}       # 默认会 rm -rf /lain/app，如果有不能删除的目录，比如 node_modules，请写在这里
                                # 为相对于 /lain/app 的路径
  script:                         # 定义构建 build image 时需要的 script
    - {BUILD_SCRIPT}
  build_arg:                    # 可选，docker build --build-arg
    - {ARG}={VALUE}
  volumes:                      # 可选
    - {ABSOLUTE_PATH}           # lain build by docker run -v `pwd`/.lain-cache/{ABSOLUTE_PATH}:{ABSOLUTE_PATH}

release:                        # 描述如何构建应用 release image，可选
  dest_base: {DEST_BASE_IMAGE}  # release image 基于的 base image
  copy:                         # 定义将哪些内容 copy 到 release 镜像中
    - src: {SRC_FILE/SRC_FOLDER}
      dest: {DEST_FILE/DEST_FOLDER}

test:                           # 描述如何构建应用 test image
  script:                       # 定义构建 test image 时需要的 script
    - {TEST_SCRIPT}

proc.{PROC_NAME}:           # 定义一个 proc, 定义 web 时，可以只用 web 表示 web.web
  user: root                # 默认为 root，还可以定义一些 release image 中存在的 user，比如 nobody 等
  type: worker              # 默认为 worker，还包括 web, portal(web 类型的 proc 会由 webrouter 导入 HTTP 流量)
  image: {PROC_IMAGE}       # 默认为 app image，也可以进行自定义
  entrypoint: {PROC_ENTRYPOINT}   # 启动 proc 时的入口点，支持类似于 Dockerfile
                                  # 中 ENTRYPOINT 的两种格式：
                                  # - entrypoint: ["executable","param1","param2"] (exec form)
                                  # - entrypoint: command param1 param2 (shell form)
                                  # **注意**：默认值为 []
  cmd: {PROC_CMD}           # 启动 proc 时定义的命令，支持类似于 Dockerfile 中
                            # CMD 的三种格式：
                            # - cmd: ["executable","param1","param2"] (exec form)
                            # - cmd: ["param1","param2"] (as default parameters to entrypoint)
                            # - cmd: command param1 param2 (shell form)
                            # **注意**：cmd 与 entrypoint 两者至少需要定义一个
  port: {PROC_PORT}         # proc 中服务所监听的端口
  ports:                    # 集群端口与(tcp/udp)应用的映射关系，由于以后可能扩展healthcheck，所以保留map结构,源端口范围:[9500:10000]
    9506:8000/tcp:          # 源端口:目的端口(不填与源端口一致)/协议类型(默认tcp)
    9504:                   # 
  labels:                   # 给容器打标签
    - proctype:router
  filters:                  # 容器部署规则过滤
    - affinity:proctype!=~router  #不与proctype为router的容器部署在同一节点
    - constraint:group==group1  # 部署到带有 group==group1 标签的节点
  memory: {PROC_MEMORY}     # 容器所用内存，默认为 32M
  num_instances: 1          # 部署时 proc 的个数，默认为 1
  https_only: true          # 针对 web 类型，默认为false, 是否只允许 https 访问
  container_healthcheck:    # docker1.12.1 or later 容器状态监测，容器滚动升级时等到上一个容器health后才升级下一个
    cmd: curl http://localhost/ # required 监测容器状态的命令
    options:                    # optional
      timeout: 5                # 超时时间
      interval: 30              # 检查间隔时间
      retries: 3                # 重试次数
  healthcheck: '/url'       # 针对 web 类型，提供该 url 给 tengine 进行健康检查，如果没有设置container_healthcheck，则该属性会继承container_healthcheck cmd为curl http://localhost/url,超时时间及间隔为10s，重试次数为3
  portal:                   # 如果 app 作为 service，提供服务的 proc 需要定义 portal
    allow_clients: "**"     # portal 允许的 client，默认提供给所有
    cmd: ./proxy            # 启动 portal 时的命令
    port: 4321              # portal 端口
  env:                      # 定义 proc 运行时需要使用的环境变量
    - ENVA=a
    - ENVB=b
  logs:                     # 声明应用需要落地的日志文件，被强制位于 /lain/logs 目录之下
    - {LOG_FILENAME}        # 对应 /lain/logs/{LOG_FILENAME}
  cloud_volumes:
    type: {CLOUDVOLUME-TYPE}    # 支持 single, multi 两种类型，默认为 multi 类型
    dirs:                       # 容器中需要进行写入的目录
      - /cloud
  volumes:                  # volume 文件，proc 被删时数据也不会移出，和 persistent_dirs 等效
                            # lain run的时候，在/tmp/lain/run/；lain deploy时，在/data/lain/volumes/
    - /var/log
    - /etc/nginx/nginx.conf:
        backup:                 # 定义备份策略
          schedule: "0 23 * * *"# 依次为 分，时，日，月，周，同 crontab 
          expire: "10d"         # 过期时间,数字+单位, 如10d表示10天, 10h表示10小时, 3m表示3分钟
          pre_run: {PRE_HOOK}   # pre hook, 备份执行前调用
          post_run: {POST_HOOK} # post hook, 备份结束后调用
  mountpoint:                   # 只支持 ProcType==web 的 proc(默认已分配 ${appname}.${LAIN-domain} 的域名)
    - a.external.domain1/b/c    # 响应 a.external.domain1 这个外网域名 b/c 段 location 的请求，转发到 lain 内网 upstream
  secret_files:                 # 定义 secret file 作为配置文件
    - /secrets/hello            # 定义文件路径，为相对路径时前面会加上 `/lain/app/`
    - secret.dat 
  secret_files_bypass: True     # 若为 True, 则当 console 拿不到 secret files 时会忽略该错误，默认为 False, 除了 layer1 应用不建议使用
  stateful: true                # 默认为 false, 表示 proc 挂掉时，并不会在另外一个节点重新启动容器
  setup_time: 0                 # 单位为秒，proc 多 instance 滚动升级时，设置升级前一个 proc 后隔多少秒升级后一个应用，用于保障服务不中断(当设置了healthcheck或者container_healthcheck时，则该参数可作为滚动升级时健康检查间隔时间)
  kill_timeout:  10             # 单位为秒，下线 proc 容器时强制删除的时间，即 docker stop timeout 时间\

use_services:       # 指出需要使用 service
  {SERVICE_NAME}:   # 标明 service app name
    - {PROC_NAME}   # 指出要使用的 service proc，这个 proc 已经定义了相应的 portal

use_resources:      # 指出需要使用 resource
  {RESOURCE_NAME}:  # 依赖的 resource app name
    memory: 32M     # 给一系列可自定义的参数赋值，也可以使用 resource 的默认值
    services:       # 定义一系列依赖的 resource proc, proc 已经定义了相应的 portal
      - {PROC_NAME}

```


## 2. lain.yaml 编译过程

用户在使用 lain-cli 命令时，lain.yaml 在各阶段的 image 的编译与转换过程如下图所示：

![workflow](img/workflow.png)


## 3. 特别选项说明

### <a id="mountpoint"></a>3.1 mountpoint

LAIN 中默认使用域名 `lain.local` 进行内部访问，除此之外也可以通过设置 `extra_domain` 来支持其它域名，但是这些都应该是对内服务能够访问的域名。

对于需要对外网提供服务的 web 类型的 proc，lain.yaml 可支持 `mountpoint` 这个属性，根据这个属性 LAIN 中的 webrouter 会将对应 server_name 的对应 location 代理到这个 proc，这个域名才应该是 proc 提供给外网访问的域名。

- `procType==web && procName==web`

```yaml
web.web:
  cmd: {PROC_CMD}   # 默认为 image 定义的 CMD
  mountpoint:       # `可选项`，系统会对 web.web 这个特殊的 proc 自动追加 {APPNAME}.{DOMAIN} 的请求响应
    - a.external.domain1/b/c   # 响应 a.external.domain1 这个外网域名 b/c 段 location 的请求，转发到 lain 内网这个 proc 对应 upstream
    - d.external.domain2/e     # 响应 d.external.domain2 这个外网域名 e 段 location 的请求，转发到 lain 内网这个 proc 对应 upstream
    - /m                       # 响应 {APPNAME}.{DOMAIN} 这个内部域名 `m` 段 localtion 的请求，转发到 lain 内网这个 proc 对应 upstream
```

- `procType==web && procName!=web`

```yaml
web.admin:
  cmd: {PROC_CMD}  # 默认为 image 定义的 CMD
  mountpoint:      # `必填项`，web 类型但是名字不是 web 的 proc 必须填写一个以上的 mountpoint
    - a.external.domain1/b/c   # 响应 a.external.domain1 这个外网域名 b/c 段 location 的请求，转发到 lain 内网这个 proc 对应 upstream
    - d.external.domain2/e     # 响应 d.external.domain2 这个外网域名 e 段 location 的请求，转发到 lain 内网这个 proc 对应 upstream
    - /admin                   # 响应 {APPNAME}.{DOMAIN} 这个内部域名 `admin` 段 localtion 的请求，转发到 lain 内网这个 proc 对应 upstream
```

### 3.2 cloud_volumes

使用 cloud_volumes 之前需要跟管理员确认 LAIN 中已经在相关目录挂载了提供 POSIX 兼容接口的分布式文件系统，如 MooseFS、CephFS 等。

如果集群中某些应用需要占用大量的磁盘空间，使用 volume 机制挂载宿主机的磁盘可能会出现空间不够的情况，这个时候可以考虑使用 cloud-volumes。此时在往 cloud_volumes 配置的文件夹中写入数据时会相应的存储到分布式文件系统中。

如果需要 proc 的多个 instance 同时共享一个目录，可以配置 cloud-volume 为 single 属性，此时所有 instance 在宿主机中对应的文件夹均默认位于 /data/lain/cloud-volumes/{APP_NAME}/{APP_NAME}.{PROC_TYPE}.{PROC_NAME}/ 目录下。

配置为 multi 属性则表示每个 instance 有个单独的文件夹可供写入，默认位于 /data/lain/cloud-volumes/{APP_NAME}/{APP_NAME}.{PROC_TYPE}.{PROC_NAME}/{INSTANCE_NO}/ 目录下。


## 4. 注意事项
在编写对应的 lain.yaml 时，有一些点需要注意：

- 在编译 prepare image 时，如果 base, script 或者 keep 发生变化，请变更这个版本号，否则不用变动

- 各阶段的 script 语句均在 /lain/app/ 路径下执行，所以上一行的 `cd` 等操作不会对下面行的工作目录产生影响

- release 阶段可以不写，如果不写，lain-cli 会把 build 阶段生成的 image 打上 release 的标签

- relese 中的 copy 指定把 script 运行结束后的中间结果 image 的路径 src 拷贝到 dest_base 中的 dest 中, src 可以是绝对路径或者是相对于 /lain/app 的路径

- 在编写 proc 时，支持几种简写形式：在定义 web 类型的 proc 时，可以直接用 web 代替 proc.{PROC_NAME}；定义 service 时，可以简单的将 portal 定义在 proc 下面，也可以单独定义一个 proctype 为 portal 的 proc

- proc 中定义的 `user` 变量必须为 release image 中已经存在的 user（可以通过 `cat /etc/passwd` 查看），如果不存在，容器启动会出现问题

- volumes 定义的文件默认位于 /data/lain/volume/{APP_NAME}/{APP_NAME}.{PROC_TYPE}.{PROC_NAME}/{INSTANCE_NO}/ 目录下

- 定义 volumes/persistent_dirs 与 stateful 选项都能使得 proc 在升级的时候不移动到其它节点，但是 statefule 定义为 true 时可以使得某个节点宕机后，集群不会在另外的节点拉起 proc，主要为了防止某些应用需要历史数据

- 由于 LAIN 中暂时不支持 proc 的别名定义机制，因此在定义 `use_services` 或 `use_resources` 时，使用的 services 和 resources 的 proc 不能存在相同的 procname，否则使用 lain-cli 进行 build 时会报错，部署后 portal 的访问也有可能出现问题。
