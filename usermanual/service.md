# Service

Lain 提供了简单的方法定义服务和服务依赖关系。

服务提供者可以在 lain.yaml 中通过定义 `service` 类型的 proc 来定义服务。
以 [demo-ipaddr-service](https://github.com/laincloud/demo-ipaddr-service.git)为例，下面定义了一个 web proc 和相应的 portal proc. (Lain 同时提供了[简写形式](#simplified-form) ):

```
appname: ipaddr-service

web:
  cmd: python service.py
  port: 10000
  num_instances: 2
  volumes:
    - /logs:
        backup_full:
          schedule: "*/5 * * * *"
          expire: 1h
    - /incremental:
        backup_increment:
          schedule: "* * * * *"
          expire: 30m

portal.portal-ipaddr:
  service_name: ipaddr
  allow_clients: "**"
  cmd: python proxy.py
  port: 10000
```

服务使用者在 lain.yaml 中通过定义 `use_services` 来描述自己依赖的服务:

```yaml
appname: ipaddr-client

use_services:
  ipaddr-service:
    - ipaddr

web:
  cmd: python client.py
  port: 10000
```

在部署 ipaddr-client 时，会自动在所有运行 ipaddr-client procs 的节点上启动一个 ipaddr-server app 的 portal proc (portal-ipaddr)，运行 ./proxy 命令，监听 10000 端口（如果不配置 portal port 的话，则和 proc port 相同）。一个节点上对每个 portal proc 最多启动一个实例，其 instance number 均为 0。

在 ipaddr-client procs 里域名 `ipaddr.ipaddr-client.lain`（或者直接用 `ipaddr`）会指向本节点的 portal proc，安全策略允许 ipaddr-client procs 连接该 portal proc 的 10000 端口。

ipaddr-server 的 ./proxy 命令会将来自客户端的请求负载均衡到各个 echo proc 上去（当然这个要看 proxy 的具体实现）。

当 ipaddr-server 更新时，会更新所有的 procs ，包括 portal procs 。新启动的 portal procs 永远会使用和被访问的 echo-server 的相同版本。


## <a id="simplified-form"></a>Portal Proc 的简写定义

如果一个 proc 的类型是 service ，如:

```yaml
service.echo:
  cmd: ./echo -p 1234
  port: 1234
  num_instances: 3
  portal:
    allow_clients: "**"
    cmd: ./proxy
    port: 4321
```

则等价于如下定义:

```yaml
proc.echo:
  cmd: ./echo -p 1234
  port: 1234
  num_instances: 3
  
portal.portal-echo
  service_name: echo
  allow_clients: "**"
  cmd: ./proxy
  port: 4321
```

即同时定义了 `proc.echo` 和 `portal.portal-echo`。在这种写法下，portal proc 的 `service_name` 默认为 service proc 的 proc name ，`port` 默认为 service proc 的 port 。


## 服务使用权限

对于服务使用者而言，只有在 `use_services` 中定义了的服务才能够访问，不能访问未定义的服务。

对于服务提供者而言，只有 appname 与该服务的 `allow_clients` 定义匹配（使用 [bash globstar](https://www.gnu.org/software/bash/manual/bash.html#The-Shopt-Builtin) 通配符语法，可以用 `**` 匹配子目录）的 app 才能访问该服务。

`allow_clients` 的值可以是一个 list，此时能够匹配上 list 中任意一个元素的 app 即可访问该服务。

如未定义 `allow_clients` ，则其值默认为 `**`。

## Resource

如果一个应用希望能够有独享的 service ，除了部署一个 service ，将其 `allow_clients` 设定为自己外，还可以使用 [Resource](resource.md) 机制实现自动复用。
