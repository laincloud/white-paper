# 第一个 LAIN 应用

> 本文会演示如何基于 LAIN 集群创建一个 LAIN 应用，它提供 HTTP 服务，当用户访问 `/` 时，返回 `Hello, LAIN.`。

## 前置条件

- 首先需要一个 LAIN 集群。如果没有的话可以参考[安装 LAIN 集群](../install/cluster.html)启动一个本地集群，建议由 2 个节点组成
- 其次需要本地的开发环境。具体步骤见[安装 LAIN 客户端](../install/lain-client.html)

> LAIN 是基于 docker 的 PaaS 系统，建议先了解下 docker 的基本概念：
> - [Docker 官方文档](https://docs.docker.com/)
> - [Docker 从入门到实践](https://yeasy.gitbooks.io/docker_practice/content/)

## 业务代码

```
package main

import (
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello, LAIN."))
	})

	http.ListenAndServe(":8080", nil)
}
```

代码的逻辑为：
- 监听 `0.0.0.0:8080` 端口
- 收到 `/` 的 HTTP 请求时，返回 `Hello, LAIN.`

## lain.yaml

`lain.yaml` 是 LAIN 应用的配置文件，如下例所示：

```
appname: hello-world  # 应用名，在集群内唯一，由小写字母、数字和 `-` 组成，且开头不能为数字，不能有连续的 `-`

build:  # 描述如何构建 hello-world:build-${git-committer-date}-${git-commit-hash} 镜像
  base: golang:1.8  # 基础镜像，类似于 Dockerfile 里的 FROM
  script:
    - go build -o hello-world  # 编译指令，类似于 Dockerfile 里的 RUN，WORKDIR 为 /lain/app

proc.web:  # 定义一个 proc，名字为 web
  type: web  # proc 类型为 web（LAIN 会为 web 类型的 proc 配置 ${appname}.${LAIN-domain} 的域名，对外提供 HTTP 服务）
  cmd: /lain/app/hello-world  # 因为 WORKDIR 为 /lain/app，所以编译好的程序在 /lain/app 目录下
  port: 8080  # hello-world 监听的端口
```

因为我们需要提供 HTTP 服务，所以定义一个 `web` 类型的 proc，LAIN 集群会为 web 类型的 proc 自动分配 ${appname}.${LAIN-domain} 的域名。

> proc.type 为 `web` 时，其名字也必须为 web，即一个 app 只能有一个 web 类型的 proc，且其名字为 web。

[laincloud/hello-world@basic](https://github.com/laincloud/hello-world/tree/basic) 的完整代码在这里。

## 本地运行

```
[vagrant@lain ~]$ cd ${hello-world-project}  # 进入工程目录

[vagrant@lain hello-world]$ lain build  # 构建 hello-world:build-${git-committer-date}-${git-commit-hash} 镜像，生成编译结果
>>> Building meta and release images ...
>>> found shared prepare image at remote and local, sync ...
>>> generating dockerfile to /Users/bibaijin/Projects/go/src/github.com/laincloud/hello-world/Dockerfile
>>> building image hello-world:build-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005 ...
Sending build context to Docker daemon 6.656 kB
Step 1/4 : FROM registry.lain.local/hello-world:prepare-0-1494908044
 ---> 7406706a7f21
Step 2/4 : COPY . /lain/app/
 ---> 45f6215362ad
Removing intermediate container 41e822d3b086
Step 3/4 : WORKDIR /lain/app/
 ---> 75c0f3094b6e
Removing intermediate container 24065cf1d7de
Step 4/4 : RUN ( go build -o hello-world )
 ---> Running in 43cefd489608
 ---> 644f596f83c8
Removing intermediate container 43cefd489608
Successfully built 644f596f83c8
>>> build succeeded: hello-world:build-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005
>>> tag hello-world:build-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005 as hello-world:release-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005
>>> generating dockerfile to /Users/bibaijin/Projects/go/src/github.com/laincloud/hello-world/Dockerfile
>>> building image hello-world:meta-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005 ...
Sending build context to Docker daemon 6.656 kB
Step 1/2 : FROM scratch
 --->
Step 2/2 : COPY lain.yaml /lain.yaml
 ---> cfdb9c518f0d
Removing intermediate container ab94a3603b8a
Successfully built cfdb9c518f0d
>>> build succeeded: hello-world:meta-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005
>>> Done lain build.

[vagrant@lain hello-world]$ lain run web  # 在本地运行
>>> run proc hello-world.web.web with image hello-world:release-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005
59f7fe4b1a7c6214361ecd5e06b19023ab7e02058888aa625749028af7b92954
>>> container name: hello-world.web.web
>>> port mapping:
>>> 8080/tcp -> 0.0.0.0:32769
```

> - lain-cli 的所有命令均需要在包含 lain.yaml 文件的目录下运行。
> - lain-cli 依赖于 git 管理版本，所以要先安装 git，而且在 `lain build` 之前进行 `git commit`

`docker ps` 时，可以看到：

```
CONTAINER ID        IMAGE                                                                     COMMAND                  CREATED             STATUS              PORTS                     NAMES
59f7fe4b1a7c        hello-world:release-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005   "/lain/app/hello-w..."   27 seconds ago      Up 31 seconds       0.0.0.0:32769->8080/tcp   hello-world.web.web
```

上面的输出表示 lain 通过 docker 把 hello-world.web.web 容器里的 8080 端口映射到了主机的 32769，所以，可以在主机上访问：

```
[vagrant@lain hello-world]$ curl http://localhost:32769
Hello, LAIN.
```

得到了预期的结果。

## 部署到 LAIN 集群

从上一小节可以看到，本地运行没有问题，现在可以部署到 LAIN 集群了：

```
[vagrant@lain ~]$ cd ${hello-world-project}  # 进入工程目录

[vagrant@lain hello-world]$ lain build  # 构建 hello-world:build-${git-committer-date}-${git-commit-hash} 镜像，生成编译结果

[vagrant@lain hello-world]$ lain tag local  # 类似于 docker tag，为 hello-world:(meta/release)-${git-committer-date}-${git-commit-hash} 镜像添加仓库前缀
>>> Taging meta and relese image ...
>>> tag hello-world:meta-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005 as registry.lain.local/hello-world:meta-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005
>>> tag hello-world:release-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005 as registry.lain.local/hello-world:release-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005
>>> Done lain tag.

[vagrant@lain hello-world]$ lain push local  # 类似于 docker push，将镜像推送到 LAIN 集群
>>> Pushing meta and release images ...
>>> pushing image registry.lain.local/hello-world:meta-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005 ...
The push refers to a repository [registry.lain.local/hello-world]
1a4886bd9611: Layer already exists
meta-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005: digest: sha256:daed70190af5fa980d6963fd3a6350591708c1568e180fe85e7eb6cfdd12d998 size: 524
>>> pushing image registry.lain.local/hello-world:release-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005 ...
The push refers to a repository [registry.lain.local/hello-world]
1a2245680fe1: Layer already exists
dfe083dd50ba: Layer already exists
edac683c8e67: Layer already exists
0372f18510d4: Layer already exists
c0b53d6ac422: Layer already exists
bcf20a0a17f3: Layer already exists
9d039e60afe3: Layer already exists
a172d29265f3: Layer already exists
e6562eb04a92: Layer already exists
596280599f68: Layer already exists
5d6cbe0dbcf9: Layer already exists
release-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005: digest: sha256:1cea69b6ed882fcc16f1f5661b3830a8b3f20263264c51d0610b8ec09e72a439 size: 2626
>>> Done lain push.

[vagrant@lain hello-world]$ lain deploy local  # 将应用部署到 LAIN 集群
>>> Begin deploy app hello-world to local ...
upgrading... Done.
>>> app hello-world deploy operation:
>>>     last version: 1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005
>>>     this version: 1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005
>>> if shit happened, rollback your app by:
>>>     lain deploy -v 1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005
```

> - local 为 LAIN 集群的名字，请参考[安装 LAIN 客户端](../install/lain-client.html)中的设置
> - `lain tag` 为镜像添加仓库前缀，之后才能进行 `lain push`
> - `release` 镜像包含了编译成果，将来会以这个镜像为基础运行容器
> - `meta` 镜像包含 `lain.yaml` 文件，用于 LAIN 集群解析，用户不需要关心
> - 部署的过程是一个异步的过程，在 `lain deploy local` 之后可以使用 `lain ps local` 查询部署结果。

此时，可以通过以下命令访问 `hello-world`：

```
[vagrant@lain hello-world]$ curl -H "Host: hello-world.lain.local" http://192.168.77.201
Hello, LAIN.
```

或者可以先更改 `/etc/hosts` 文件，然后直接使用域名访问：

```
[vagrant@lain hello-world]$ echo "192.168.77.201 hello-world.lain.local" >> /etc/hosts
[vagrant@lain hello-world]$ curl http://hello-world.lain.local
Hello, LAIN.
```

> 上面的 `192.168.77.201` 是本地 LAIN 集群的虚拟 IP，没有以 `vip` 方式启动，请使用 `192.168.77.21`

得到了 `Hello, LAIN.` 的响应，符合我们的预期。
