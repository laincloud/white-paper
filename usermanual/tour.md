##Lain Basics

####新建/迁移 一个 App 到 Lain 体系 [建议的开发流程]
######本地开发环境
建议的开发环境是使用虚拟机方案 [lain-box](https://github.com/laincloud/lain-box) ，使用 vagrant + virtualbox ，在自己的电脑上启动一个 centos7.1 的虚拟机，作为主力开发环境
具体使用方法见 [lain-box](https://github.com/laincloud/lain-box)  的 readme 或简单参见如下:

宿主机上代码目录（假设在 `/code/`）
```
cd /code/ && git clone https://github.com/laincloud/lain-box.git
cd /code/lain-box
vim config.yaml
```
按照其中的说明修改想要在宿主机和虚拟机之间映射的目录，建议将宿主机的 `hello` 目录（假设在 `/code/hello`）映射到虚拟机里的 `/apps/hello`，即写成
```
	hello: /code/hello
```
然后启动虚拟机
```
vagrant up
vagrant ssh
```
在虚拟机中配置 Lain 集群
```
lain version
lain update

lain config show
lain config save local domain lain.local
lain config save-global private_docker_registry registry.lain.local
```

#####docker 基础知识
* [Docker 官方文档](https://docs.docker.com/)           
* [Docker 从入门到实践](https://yeasy.gitbooks.io/docker_practice/content/)


###### 分析项目，创建 lain.yaml
首先根据项目的编译和发布情况，选择构建和发布的基准镜像（也就是 build.base 和 release.dest_base ），推荐  [centos-lain 镜像](https://github.com/laincloud/centos-lain)

 根据项目的线上运行启动方式，撰写镜像里的启动脚本（entry.sh）
一般的处理是每个 proc 有一个 cmd，用一个 shell 脚本来做好一些初始化然后 exec  真正的线上运行进程

根据项目的线上线下配置，撰写镜像里使用的配置文件（config）
lain 里面常用的方案是，entry.sh 里从 LAIN_DOMAIN 这个环境变量得知运行时应用所处的环境，然后传入不同的运行时参数（配置文件）

根据项目的编译方法，撰写 lain.yaml 里的 build.prepare.script 和 build.script
根据项目的发布产物，撰写 lain.yaml 里的 release.copy
根据项目运行的需要，规划应用（对应全局唯一的 appname）的 proc （一个或多个）
> `web` 其实是 web.web 的简写 [proctype].[procname] 前面是 proc 类型，后面是proc 名字
> `proctype` 可选类型包括 web/worker/portal
> `procname`  在一个app内不可重复
> `worker` 和 `web` 类型的区别是， web会由 webrouter 引入web流量
> `worker` 就是一般的后台程序
> `web.web` 是一个默认的 web proc，享有最大便利的简写，下面不用写 mountpoint 也会默认绑定一个 appname.{lain domain} 的域名流量过来 参见 mountpoint

根据运行时对内存和cpu的需求，撰写 proc.cpu / proc.memory
> 需要评估现有应用服务时占用的内存，cpu需求，在 lain.yaml 里写明
> 运行时也可以动态 scale

根据运行时需要的环境变量，撰写 proc.env
>有些应用需要环境变量，可以按照 demo lain.yaml 里的写法注入运行时的环境变量
>也可以在 entry.sh 里面动态生成

根据运行时需要保存到磁盘的数据，包括落地的日志，撰写 proc.[persistent_dirs|volumes]
>目前这些数据会从宿主机映射进去

根据运行时需要响应的内外网域名/子路径，撰写 proc.mountpoint

根据运行时程序响应的端口，撰写 proc.port
>运行时程序响应的端口，会作为 loadbalancer（包括  webrouter， portal）必须的信息，传递给他们

>然后 loadbalancer 会按照这个端口把对应域名的请求 proxy 到应用容器里

>举例： 一个 appname 为 testapp 的 proc ，定义为  web.web ，标注 port 为 8000 , 标注 mountpoint 为 test.yixin.com ，里面定义跑一个 tomcat 监听 8000 ， 那么一旦外部引流过来，webrouter 会把 Host 为 test.yixin.com:80 的请求 proxy 到 这个 proc 的 8000 端口，tomcat 就能收到这个请求并响应了

在 slack 创建一个项目对应的 channel，并设定在 notify.slack
> 例如 hello 的项目都可以使用 #hello 频道
其他

#####本地构建测试   [Lain CLI](https://github.com/laincloud/lain-cli)
`lain dashboard local`
`lain reposit local`
> 注册本 app 到 lain 集群

`lain prepare`
>本地构建和ci构建的基础，预先构建好一个可网络共享，可复用的编译依赖环境 docker image

`lain build`
>复用上述编译依赖环境，进行项目构建，并生成 docker image

`lain run ${proc_name}`
>前置条件是之前 lain build 成功

>假设 lain.yaml 里写了 web: 段，则其实是定义了类型 (proc type) 为 web ，名字 (proc name) 为 web 的 proc，可用 lain run web 在本地拉起来，容器名字 为 docs.web.web

>docker 自动将 web proc 里面定义的 port 端口映射到 lain-box，如下例子，可以在 lain-box 或者 mac 宿主机上直接访问 192.168.10.24:32778 来访问到 docs.web.web 容器内的 80 端口服务

```
vagrant@lain (master) % lain run web
>>> run proc docs.web.web with image docs:release
>>> DOCKER_HOST='' docker run -d --name=docs.web.web -w /lain/app -p 80 -e TZ=Asia/Shanghai -e LAIN_APPNAME=docs -e LAIN_PROCNAME=web -e LAIN_DOMAIN=lain.local -e DEPLOYD_POD_INSTANCE_NO=1 -e DEPLOYD_POD_NAME=docs.web.web -e DEPLOYD_POD_NAMESPACE=docs  docs:release /entry.sh
99b5afc4537c6cc075c4f6f352c5fdff2066cec3209fb5adb482bb2805b0be26
>>> container name: docs.web.web
>>> Port mapping:
>>> DOCKER_HOST="" docker port docs.web.web
>>> 80/tcp -> 0.0.0.0:32778
vagrant@lain (master) % docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                            NAMES
99b5afc4537c        docs:release        "/entry.sh"         27 seconds ago      Up 26 seconds       443/tcp, 0.0.0.0:32778->80/tcp   docs.web.web
```

`lain debug ${proc_name}`
>前置条件是上面的 lain run ${proc_name}
>其实是等于执行 docker exec -ti ${container_name} bash ，进入到容器内部
```
vagrant@lain (master) % lain debug web
>>> attach proc instance docs.web.web
>>> DOCKER_HOST='' docker exec -ti docs.web.web bash
root@99b5afc4537c:/lain/app# ps axu
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0  17964  1448 ?        Ss   08:13   0:00 /bin/bash /entry.sh
root         6  0.0  0.0  31292  2716 ?        S    08:13   0:00 nginx: master process nginx -g daemon off;
nginx        7  0.0  0.0  31664  1664 ?        S    08:13   0:00 nginx: worker process
root         8  1.0  0.0  18152  1848 ?        Ss   08:19   0:00 bash
root        21  0.0  0.0  15564  1164 ?        R+   08:19   0:00 ps axu
```

`lain stop ${proc_name}`
>前置条件是上面的 lain run ${proc_name}
 >停止这个 proc 的本地容器

`lain rm ${proc_name}`
>前置条件是 lain stop ${proc_name}
>删除清理上面已停止的 本地容器


####登录认证，权限管理
>出于安全考虑，Lain集群已开启OAuth2认证，验证开启之后不再随意允许用户对app进行deploy、scale操作。

>想要使用Lain Cli进行此类操作需要事先使用sso账户执行lain login local (没有此命令请先执行lain update)，并且要求该用户是app的maintainer。


####应用部署和运维
>上述镜像发布完成之后，就可以使用 lain 的 应用控制台 (console) 或者 命令行工具 (CLI) 部署应用

`lain CLI`

前提是需要 在 CLI 进行登录，并拥有 这个应用的 管理权限
`lain login local`   
>用在  https://sso.lain.local 注册的用户名密码进行登录

`lain deploy local`  
>会调用对应的 lain 集群 console 的 api ，升级本应用到目前最新的镜像版本。

`lain deploy  local [-v meta-version]`  
>部署应用的指定版本，注意这个版本必须是之前 lain release 过的版本

>例如 在 testapp 的目录下执行  lain deploy local -v "1448593783-4254886fabb4390cfcc1e3c7544a28f721e3e923" ，会发起 local 集群上 testapp 应用的更新到指定版本

`lain scale local $procname -n / -c / -m `
>会调用对应的 lain 集群 console 的 api ，设置本应用的某个proc的实例个数，cpu个数，内存分配量（容器是会全部重建的）
>
>例如 在 testapp 的目录下执行  lain scale local web -n 2 ，会发起 local 集群上 testapp 应用 web 这个 proc 保持 2个后端容器进程的命令

>例如 在 testapp 的目录下执行  lain scale local web -m 600M ，会发起 local 集群上 testapp 应用 web 这个 proc 每个容器进程分配内存设置为 600M 的命令

线上日志查询/线上debug
`lain enter`
>如： lain enter local web 1
>>local:   部署环境
>>web:    部署的服务名字
>>1:         对应的实例编号

>标准输入输出的日志查询  参见 用户手册中的日志章节


#####监控告警
Lain 提供了开箱即用的监控告警系统，[hedwig](https://github.com/laincloud/hedwig)
######默认监控项
每一个 proc 启动会由 lain 集群自动开启 cpu/mem/network 的相关监控
举例： appname 为 hello 的应用里定义一个 类型为 web，名字为 web1 的 proc （一般写为 web.web1 ），则在hello这个应用部署到某个集群之后（假设为 local 集群），可以在所在集群的监控系统中自动生成监控

监控系统 grafana 地址: `hedwig-grafana.lain.local`

每一个 web 可访问的 域名（mountpoint）会由 lain 集群自动开启 QPM/RST/EPM 的相关监控
>举例： appname 为 hello 的应用里定义一个 类型为 web， 名字为 web 的proc （一般写为 web ），写了一个 mountpoint ，hello.abc.com ，则在hello这个应用部署到某个集群之后（假设为 local 集群），对应监控系统会生成2个域名的 QPM/RST/EPM 监控

>hello.local.in ，这个域名是集群里默认给的域名
>
>hello.abc.com ，这个域名是写了 mountpoint 的结果
