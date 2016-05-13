# Service 与 Resource

[service](../../usermanual/service.md) 与 [resource](../../usermanual/resource.md) 是 LAIN 中重要的设计，在 lain.yaml 中定义了 portal 类型的 proc 则被 LAIN 视为一个 service。resource 则是一种 private service。

在 LAIN 中，console 提供了各种对于 app 的操作，service 与 resource 类型的 app 如何 部署也是由 console 定义的。

开发者如果想要进行定制，或者增加新的 app 类型，可以对 console 的代码进行调整。现有的主要的 service 与 resource 部署规则如下：

- resource 部署之后并不会创建相应的容器，只会在 LAIN 中进行记录，当有使用 resource 的 client app 被部署后，才会根据 resource 的配置生成相应的 resource instance； 

- client app 部署时依赖某些 service 与 resource，如果 resource 不存在，client app 会部署失败，并且不会创建相应的容器；但是如果只是 service 不存在，client app 会被部署起来，等待部署相应的 service；

- service 之间允许循环依赖， 也就是说 Service A 可以使用 Service B，而 Service B 也可以使用 Service A，两者的部署顺序并不会影响 app 被部署起来。但是 LAIN 并不推荐这种相互依赖的设计；

- resource 可以再依赖 service，但是不能再依赖于另外的 resource。同样不推荐这种设计，建议 resource 设计为可以公共使用的基础资源；


另外，console 与 deployd 在处理 service 与 client app 时采用如下规则：

1. 部署 service app 时，console 会调用 deployd 创建相应的 podgroup 以及 depends；deployd 随后创建相应 worker 容器，并记录 portal 相关信息；

1. 部署 client app 时，console 会在传给 deployd 的 PodGroupSpec 中加入 depends 信息；deployd 随后创建 client app 的 worker 容器，并利用 service 记录的 portal 信息创建 相应的 portal；
