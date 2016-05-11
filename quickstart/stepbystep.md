# 基于 Lain 集群创建一个全新的应用

>本文会演示如何基于 Lain 集群创建一个新应用，该应用能够验证此时此刻是否能够成功访问  404 网站，并将结果记录在日志文件中。

### 有何不同

创建基于 Lain 集群的应用，整体上的流程

- 规划/更新 lain.yaml 配置
- 编写业务代码
- 创建/更新持续集成任务
- 滚动部署到本地/开发/产品环境
- 回到 1 或者 2 

### 基础：lain.yaml

与创建一个运行于机器（物理机/虚拟机）的“野生”应用相比，基于 Lain 集群的创建工作所增加的无非是一个应用内全局的配置文件，一般我们将它放置在应用的根目录下，称之为 lain.yaml.

lain.yaml 主要规定/描述了如下几件事情
- 为应用起一个好听的名儿
- 基础技术栈
- 构建过程
- 微服务拆分以及内部配置

以上几步描述了一个应用的配置，Lain 集群根据此为应用调度资源，创建实例并运行。

为此，有必要了解一些 Lain 的基本概念，和与之相对的 lain.yaml 中的配置

### Lain 基本概念

- App，运行于 Lain 集群中的一个应用，很显然对应现实中的一个应用，比如一个你自己的博客站点。一个 lain.yaml ，即是对一个 App 的描述。记得为应用起一个好听且 Lain 集群内唯一的名字
- Proc，一个 Lain App 包含若干自己的 Proc，对应于拆分出来的微服务。Proc 有自己 App 内唯一的名字，和类型。比如我们可能把博客站点，划分成一个前端 web 服务，和一个后端 db 服务。

App 和 Proc，分别对应于完整应用和微服务，就形成了 Lain 上应用的骨架。


### 开始创建应用

回到开篇提到的翻墙监控应用，我们来一步一步完成它。

先给应用取个名字，在 lain 集群中，每个应用的名字是全局 unique 的，字符集要求是  小写字母以及数字，非连续 "-" 符号，且开头不能为数字，比如起名为 anti-gfw

appname 一旦注册到 Lain 集群，且这个集群开启了 auth，那么这个 appname 就会被你占据和管理，除非你把别人加入这个 app 的管理组，否则任何人不能控制该 app。

### 挑选技术栈

选用 golang 完成 DEMO 项目，基础镜像可自行任意选取符合自己需求的。

### 构建过程

golang 项目的构建采用 go 命令行工具，请参考相关文档。

### 规划应用服务

anti-gfw App 功能简单明白，只需要规划一个类型为 worker 的 Proc 即可，起名为 gfw-watcher。worker 类型的 Proc 可以理解为由 Lain 集群守护的 daemon 程序，由集群运行指定的 cmd 命令，且保证实例始终存活。

至此，lain.yaml 有了大概的轮廓

```
appname: anti-gfw
build:
    base: sunyi00/centos-golang-nodejs-python:3.0.1
    script:
        - go build -o anti-gfw
proc.gfw-watcher:
    type: worker
    cmd: /lain/app/entry.sh
```

我们定义了一个 anti-gfw 应用，因为是 golang 技术栈，选用了推荐的基础镜像，并且确定了构建方法。

同时梳理了应用内唯一服务，即一个 worker 类型的 Proc，该 Proc 实例由集群守护，始终执行着既定的任务。

接下来需要做的事情就是 
- 完成应用业务代码
- 根据具体需求，完善配置细节
- 调试，ci，部署等

应用业务代码

业务需求主要包括
- 设置 http_proxy
- 访问 404 网站
- 记录结果到日志文件

具体业务代码请参考 gfw-watcher  # TODO 附 demo 代码 repo 到 github

### 完善 lain.yaml

#### 构建阶段

build 阶段除了常规的构建命令 script 之外，可以添加可选的 prepare 子阶段，用来缓存构建结果，供 lain debug/build 使用。

```
prepare:  # lain prepare 时，在 docker container 的 /lain/app/ 下执行下列命令，这一步的结果会默认缓存，以备 lain debug|build 使用，可用来安装稳定的系统依赖
    version: 0 # prepare 版本用 `build.prepare.version` 来定义，内容为字符串，base, script 或者 keep 发生变化时，请变更这个版本号，否则不用变动
    script: # 在 prepare 构建时于 /lain/app/ 路径下执行的顺序指令，注意这里类似 Dockerfile 的 RUN 语法，上一行的 `cd` 等操作不会对下面行的 `PWD` 产生影响
      - echo "prepare"
```

#### 规划编译产出

build 阶段完成构建，亦即编译工作。由于 build 阶段使用的镜像以及 build 过程中的操作，可能产生大量不需要带到生产环境的中间结果。

可以定义一个可选的 release 阶段，仅将需要的编译产出打包成干净的生产环境镜像。

例如

```
release: # 可以不写，如果不写，就把build阶段生成的image打上release的标签
    dest_base: centos:1.0.1   # release image基于的base image，如果不写，就把script执行完生成的中间结果image直接作为release image
    copy:     # 指定把script运行结束后的中间结果image的路径src拷贝到dest_base中的dest中, src 可以是绝对路径或者是相对于/lain/app的路径，拷贝之前，dest image中会自动创建/lain/app空目åease的标签
        - src: /lain/app/entry.sh
          dest: /lain/app/entry.sh
        - src: /lain/app/watcher
          dest: /lain/app/watcher
# 如果不写src, dest，比如只写 - bin/*， 那么相当于把 src image 中的 bin/* 拷贝到 dest image 中的相同路径下
```

#### 给 proc 注入其他配置

Proc 除了必备的 name, type 以及 cmd 之外，也有其他可选的配置。基于 DEMO 的需求，可能添加的其他配置

```
proc.gfw-watcher:    # 定义一个 Proc，且指定 name 为 gfw-watcher
    type: worker     # 指定 Proc type 为 worker
    num_instances: 1 # 指定 Proc 在集群中的实例数量。默认为 1
    cmd: /lain/app/entry.sh # 运行的命令
    logs:            # logs 字段为一个文件列表，用来声明应用需要落地的日志文件。 注意这里的日志文件只能是单纯的文件名，且被强制位于 /lain/logs 目录之下
        - monitor.log # 对应 /lain/logs/monitor.log
```

logs 字段用来声明一个日志文件列表，注意这里的文件列表只能是单层级的文件名，例如 info.log, monitor.log，不能含有路径。

在 logs 中声明的日志，最终会被 Lain 集群收集到 Kafka 集群中，给予默认7天的保存。日志的收集和检索请查看。

在 kafka 中的 topic 名格式是： appname.proctype.procname.logs.[日志文件名]，比如 anti-gfw.worker.gfw-watcher.logs.monitor.log

#### 最终的 lain.yaml

```
appname: anti-gfw
build:
    base: centos-golang-nodejs-python:2.0.1
    script:
        - go build -o watcher
release:
  dest_base: centos:1.0.1
  copy:
    - src: /lain/app/entry.sh
      dest: /lain/app/entry.sh
    - src: /lain/app/watcher
      dest: /lain/app/watcher
proc.gfw-watcher:
    type: worker
    num_instances: 1
    cmd: /lain/app/entry.sh
    logs:
        - monitor.log
notify:
    slack: "#anti-gfw"
```
 
#### 本地调试 proc

[lain-box](https://github.com/laincloud/lain-box) 用来在本地启动一个可用的 lain in vagrant 环境。

- 查看目前配置 `lain config show`
- 保存 local 配置  `lain config save local domain lain.local`

vagrant ssh 进入虚拟机中，进入 gfw-watcher 代码目录

- lain dashboard local     
- lain reposit local      # 往 local 环境注册本 app
- lain prepare
- lain build
- lain run gfw-watcher

顺利的话，即在本地启动了一个 Proc 

docker ps 可以看到

```
[vagrant@lain gfw-watcher]$ docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED             STATUS              PORTS               NAMES
eeffbc704a4f        anti-gfw:release    "/lain/app/entry.sh"   3 seconds ago       Up 2 seconds                            anti-gfw.worker.gfw-watcher
```

运行 lain debug gfw-watcher 可以以交互方式进入实例中进行 debug


#### 部署应用

>上述 lain build 成功之后

- lain tag local
- lain push local
- lain deploy local

部署的过程是一个异步的过程，在 `lain deploy local ` 之后可以使用 `lain ps local` 查询部署结果
