# Proc 

Proc 是在 [lain.yaml](../../usermanual/lainyaml.md) 中定义的 app 的基本单元，目前支持 web, worker, portal 三种类型的 proc。

LAIN 中部署的容器都是基于 proc 中定义的内容而创建，共有以下组件会对 proc 进行处理：

- [lain-sdk](https://github.com/laincloud/lain-sdk) 提供对 lain.yaml 的解析方案，生成 LainConf 对象；

- [console](https://github.com/laincloud/console) 根据 lain-sdk 的解析结果进行封装，生成对应的 PodGroupSpec 传给 deployd；

- [deployd](https://github.com/laincloud/deployd) 根据 console 传入的结果，将 PodGroupSpec 中对应的消息进行解析，并作为参数传给 docker swarm；

- docker swarm 接收到参数之后调用 docker 进行部署；

由此，开发者如果想要在 LAIN 中增加 proc 的属性，需要对 lain-sdk，console 以及 deployd 三者进行调整。

**注意：** lain-sdk 中默认使用 python 编写的 parser 进行解析，但是也提供了 lua 版本的解析器。开发者可以在了解了 lua parser 后，将 python parser 进行替换。

