# Resource

对于某些服务，应用会需要自己独享而不是共享，以减少应用间资源冲突（比如一个 redis 实例，或者一个 [demo-ipaddr-resouce](https://github.com/laincloud/demo-ipaddr-resource.git) 实例）。除了手工部署服务并通过 allow_clients 来限定使用权限外，lain 提供了 resource 的概念，供应用部署时自动从模板生成新的服务，供应用使用。

资源提供者在 lain.yaml 里声明自己是 resource 。resource 类型的 app 在部署时只执行注册版本操作，并不会真正运行 proc 。

```yaml
appname: ipaddr-resource
apptype: resource

web:
  cmd: python service.py
  port: 10000

portal.portal-ipaddr-r:
  service_name: ipaddr-r
  allow_clients: "**"
  cmd: python proxy.py
  port: 10000
```

资源使用者使用 `use_resources` 定义要用的资源:

```yaml
appname: ipaddr-client
use_resources:
  ipaddr-resource:
    services:
      - ipaddr-r
        
web:
  cmd: python client.py
  port: 10000
```

当应用 ipaddr-client 部署时，会自动生成一个 appname 为 `resource.ipaddr-resource.ipaddr-client` 的应用。在生成应用时，将 ipaddr-resource 应用的 lain.yaml 看作是一个 [Jinja2](http://jinja.pocoo.org/) 模板，用可选的 args 参数去渲染，得到新应用的 lain.yaml 。新渲染得到的 lain.yaml 还会做两个改动:

1. appname 会自动变为 `resource.ipaddr-resource.ipaddr-client` ，即由 `resource`、resource appname 和使用 resource 的应用的 appname 组合而成）；
2. 去掉 `apptype: resource`

Lain 会使用改动之后的 lain.yaml 作为应用配置部署该应用。

使用该 resource 的应用（本例中为 ipaddr-client）被称为 *资源所有者 (Resource Owner)* 。

如果该应用已经在集群中部署过，则不会执行这个逻辑。即：资源所有者升级不会带来专属资源的升级。

如果在 `use_resources` 中定义了 `services`，则资源应用提供的服务可以被资源所有者调用，其机制与 [Service](service.md) 相同。


## Resource 的升级

Resource 应用虽然是在部署资源所有者时动态生成的，但其管理权限仍然归 resource 应用的维护者。在 [App Console](appconsole.md) 中，resource 的维护者可以看到集群中部署的各个 resource 应用实例，对于每一个应用实例，可以分别的进行管理和升级。
