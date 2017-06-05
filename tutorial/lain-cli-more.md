# lain-cli 的更多操作

> 在了解了如何构建一个 LAIN 应用之后，我们可以了解一下 lain-cli 的更多操作。

## 前置条件

- 首先需要一个 LAIN 集群。如果没有的话可以参考[安装 LAIN 集群](../install/cluster.html)启动一个本地集群，建议由 2 个节点组成
- 其次需要本地的开发环境。具体步骤见[安装 LAIN 客户端](../install/lain-client.html)
- 还需要一个 LAIN 应用。如果没有的话可以参考[第一个 LAIN 应用](first-lain-app.html)创建

> 以下均假定在本地开发环境执行过以下操作：
>
> ```
> lain config save local domain lain.local
> ```
>
> 即 lain-cli 配置了一个 LAIN 集群，其域名为 `lain.local`，在本地保存的名字为 `local`。

## 权限管理（可选）

LAIN 提供了认证和授权体系，统称为 [auth](../adminmanual/auth.html)。
在生产环境中，我们一般会开启 LAIN 集群的 `auth` 系统。这时，利用 `lain-cli`
对 LAIN 集群进行操作的时候，需要先在 https://sso.lain.local/spa/ 里注册账户，
然后在本地开发环境配置 sso 地址：

```
[vagrant@lain hello-world]$ lain config save local sso_url https://sso.lain.local
```

然后登录：

```
[vagrant@lain hello-world]$ lain login local
```

## 本地调试

```
[vagrant@lain hello-world]$ lain build  # lain-cli 依赖于 git 管理版本，lain build 之前请先 git commit
[vagrant@lain hello-world]$ lain run ${proc_name}
```

> 假设 lain.yaml 里写了：
>
> ```
> proc.web:
>   ...
> ```
>
> 表示是定义了名字 (proc_name) 为 web 的 proc，可用 lain run web 在本地拉起来，容器名字 为 ${appname}.${proc_type}.web

这时 `docker ps`，可以看到：

```
CONTAINER ID        IMAGE                                                                     COMMAND                  CREATED             STATUS              PORTS                     NAMES
59f7fe4b1a7c        hello-world:release-1496631978-a46ef7afcbbc18c1c1248bcbb84fa770d0758005   "/lain/app/hello-w..."   27 seconds ago      Up 31 seconds       0.0.0.0:32769->8080/tcp   hello-world.web.web
```

`PORTS` 一列表示 lain-cli 利用 docker 把容器的 `8080` 端口映射到了主机的 `32769` 端口，即我们可以通过以下方式访问 `hello-world` 服务：

```
[vagrant@lain hello-world]$ curl http://127.0.0.1:32769
Hello, LAIN.
```

此外我们还可以进入容器：

```
[vagrant@lain hello-world]$ lain debug web
```

> 等价于 `docker exec -ti ${container-name} bash`

调试完之后，我们需要关闭并删除本地启动的容器，以免影响下次运行：

```
[vagrant@lain hello-world]$ lain stop web
[vagrant@lain hello-world]$ lain rm web
```

## 部署一个 LAIN 应用的完整流程

部署一个新的 LAIN 应用的时候，其实需要先把这个应用注册到 LAIN 集群：

```
[vagrant@lain hello-world]$ lain reposit local
```

> 如果没有执行 `lain reposit`，`lain deploy` 的时候会先 `reposit`，再 `deploy`。

如果 `lain.yaml` 里有 `build.prepare` 字段的话，需要先 `prepare`：

```
[vagrant@lain hello-world]$ lain prepare local
```

> 如果 `lain.yaml` 里有 `build.prepare` 字段且没有执行 `lain prepare`，`lain build` 的时候会先 `prepare`，再 `build`。

然后编译：

```
[vagrant@lain hello-world]$ lain build  # lain-cli 依赖于 git 管理版本，lain build 之前请先 git commit
```

接着打上版本标签：

```
[vagrant@lain hello-world]$ lain tag local
```

推送到 LAIN 集群仓库：

```
[vagrant@lain hello-world]$ lain push local
```

然后就可以部署了：

```
[vagrant@lain hello-world]$ lain deploy local
```

> `lain deploy` 默认部署最新版本，`-v` 参数可以回滚到以前部署过的版本，
> 比如 `lain deploy local -v 1495541505-9476383ce62201aedd92eaa33467cb19bb8f01ba`

## 线上运维

### 查看当前运行状态

```
[vagrant@lain hello-world]$ lain ps local
```

### 改变 proc 的实例个数

比如将 `web` `proc` 改为 2 个实例：

```
[vagrant@lain hello-world]$ lain scale -n 2 local web
```

### 改变 proc 的预留内存

比如将 `web` `proc` 的预留内存改为 64 M：

```
[vagrant@lain hello-world]$ lain scale -m 64M local web
```

### 进入容器

```
[vagrant@lain hello-world]$ lain enter local web 1
```

> `local` 为 LAIN 集群的名字，`web` 为 proc name，`1` 是实例的序号

### 观察容器的标准输入输出

```
[vagrant@lain hello-world]$ lain attach local web 1
```

> - `local` 为 LAIN 集群的名字，`web` 为 proc name，`1` 是实例的序号
> - 可以在命令后加 Linux 管道过滤，比如 `lain attach local web 1 | grep ERROR`

### 查看本人所管理的应用

```
[vagrant@lain hello-world]$ lain dashboard local
```
