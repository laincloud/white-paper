# Resource

对于某些服务，应用会需要自己独享而不是共享，以减少应用间资源冲突（比如一个 redis 实例，或者一个 [demo-ipaddr-resouce](https://github.com/laincloud/demo-ipaddr-resource.git) 实例）。除了手工部署服务并通过 allow_clients 来限定使用权限外，lain 提供了 resource 的概念，供应用部署时自动从模板生成新的服务，供应用使用。

资源提供者在 lain.yaml 里声明自己是 resource 。resource 类型的 app 在部署时只执行注册版本操作，并不会真正运行 proc 。

```yaml
appname: ipaddr-resource
apptype: resource

web:
  cmd: python service.py
  memory: "{{ memory|default('64M') }}"
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
    memory: 32M
    services:
      - ipaddr-r
        
web:
  cmd: python client.py
  port: 10000
```

当应用 ipaddr-client 部署时，会自动生成一个 appname 为 `resource.ipaddr-resource.ipaddr-client` 的应用。在生成应用时，将 ipaddr-resource 应用的 lain.yaml 看作是一个 [Jinja2](http://jinja.pocoo.org/) 模板，用可选的 args 参数去渲染，得到新应用的 lain.yaml。新渲染得到的 lain.yaml 还会做两个改动:

1. appname 会自动变为 `resource.ipaddr-resource.ipaddr-client` ，即由 `resource`、resource appname 和使用 resource 的应用的 appname 组合而成）；
2. 去掉 `apptype: resource`

Lain 会使用改动之后的 lain.yaml 作为应用配置部署该应用。

使用该 resource 的应用（本例中为 ipaddr-client）被称为 *资源所有者 (Resource Owner)* 。

如果该应用已经在集群中部署过，则不会执行这个逻辑。即：资源所有者升级不会带来专属资源的升级。

如果在 `use_resources` 中定义了 `services`，则资源应用提供的服务可以被资源所有者调用，其机制与 [Service](service.md) 相同。


## Resource 的升级与管理

Resource 应用虽然是在部署资源所有者时动态生成的，但其管理权限除了 resource owner 之外也包括 resource 应用的维护者。在 resource 的 git 目录下，resource 的维护者可以使用 `lain ps` 看到集群中部署的各个 resource 应用实例。对于每一个应用实例，也可以通过 lain-cli 分别进行管理和升级。

在升级时也会遵守一些规定：

### <a id="upgrade-resource-owner"></a>升级 Resource Owner

升级 resource owner 时：

* 如果 `use_resources` 配置引入了新的 resource app ，则会在升级 resource owner **之前**先实例化该 resource 。

* 如果 `use_resources` 配置删除了一个之前在用的 resource app ，则会在升级 resource owner **之后**将对应的 resource instance app 下线。

* 如果 `use_resources` 配置中的 resource app 的内容发生了更改，**不会**自动引起 resource instance app 的重新部署，必须由 resource app 的维护者显式更新。

### <a id="upgrade-resource-app"></a>升级 Resource App

Resource App 部署时不会执行容器创建，只会在 LAIN 中进行记录。升级 Resource App 时，同样只更新 LAIN 中记录，不会对已经实例化的 resource instances 做任何操作。

如果要更新 resource instance ，则需要按照 [Upgrade Resource Instance](#upgrade-resource-instance) 所述执行。

### <a id="upgrade-resource-instance"></a>升级 Resource Instance

对于 resource 应用实例，维护者可以在 lain-box 中使用 lain-cli 进行管理，通过`lain deploy/scale/undeploy -t {{ resource instance name}}` 来进行管理。

**提示：** `-t` 参数允许不在某个特定 app 的工作目录下对其它的某个 app 进行部署相关操作。


## 注意事项

- 由于 LAIN 中暂时不支持 proc 的别名定义机制，因此在定义 `use_services` 或 `use_resources` 时，使用的 services 和 resources 的 proc 不能存在相同的 procname，否则使用 lain-cli 进行 build 时会报错，部署后 portal 的访问也有可能出现问题。

- resource 可以依赖于某个 service，但是不能再依赖于另外的 resource。但是不推荐 resource 依赖 service 的这种设计，建议 resource 设计为可以公共使用的基础资源；
