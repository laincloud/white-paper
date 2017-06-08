# LAIN 上应用构建部署示例

>在建立了 Lain 上的 App 和 Proc 概念后，
本节将一步步演示如何在 lain-box 利用 laincli 构建、部署应用。

## 准备

- 首先需要部署一个本地集群，建议由两个节点组成。具体步骤见 [安装 LAIN 集群](../install/cluster.html)
- 其次需要本地的开发环境。具体步骤见[安装 LAIN 客户端](../install/lain-client.html)

## 部署 demo 应用

这里，我们将部署三个示例应用：
- [demo-ipaddr-service](https://github.com/laincloud/demo-ipaddr-service)：返回程序所在容器的 IP 列表
- [demo-ipaddr-resource](https://github.com/laincloud/demo-ipaddr-resource)：返回程序所在容器的 IP 列表
- [demo-ipaddr-client](https://github.com/laincloud/demo-ipaddr-client)：返回程序所在容器、
  `demo-ipaddr-service` 和 `demo-ipaddr-resource` 的 IP 列表

由于 client 依赖 resource 和 service，这一点可以在 client 的
[lain.yaml](https://github.com/laincloud/demo-ipaddr-client/blob/master/lain.yaml) 中看到,
所以需要在 resource 和 service 部署之后，最后部署。
更多背景知识可见 [service](../usermanual/service.html), [resource](../usermanual/resource.html), 
两者的主要作用是进行访问控制，即保证资源的安全性。

### 下载示例代码

```
cd /home/vagrant
git clone https://github.com/laincloud/demo-ipaddr-resource.git
git clone https://github.com/laincloud/demo-ipaddr-service.git
git clone https://github.com/laincloud/demo-ipaddr-client.git
```

### 部署 ipaddr-resource

```
cd /home/vagrant/demo-ipaddr-resource
lain dashboard local # 看当前 local 集群有哪些应用
lain reposit local # 注册应用
lain build # 构建应用
lain tag local # 将构建好的镜像打上 tag，时间戳＋commit id
lain push local # 将构建好的镜像 push 到本地集群
lain deploy local # 部署应用
```

resource 是没有实例的, 如果执行 `lain ps local`, 可以看到并没有容器列表.

### 部署 ipaddr-service

```
cd /home/vagrant/demo-ipaddr-service
lain dashboard local
lain reposit local
lain build
lain tag local
lain push local
lain deploy local
lain ps local #查看应用当前的部署信息
lain scale -n 2 local webc # 将 web proc 扩容为 2 个
lain ps local
```

这时候可以看到，ipaddr-service 的 proc list 已经有实例了，但是还没有 portal.

### 部署 ipaddr-client

部署过 service 后，即可部署

```
cd /home/vagrant/demo-ipaddr-client
lain dashboard local
lain reposit local
lain build
lain tag local
lain push local
lain deploy local
lain ps local # 这里需要看一下实用的 sevice 和 resource 是否已部署好
lain scale -n 2 local web # 等待上一步部署结束，就可以扩容
lain ps local
```

这时可以看到一个叫做 resource.ipaddr-resource.ipaddr-client 的应用部署起来了。
执行 `curl ipaddr-client.lain.local` 可以看到，返回 resource / service / client 三个 calico ip.

这时，执行 `lain dashboard local`, 可以看到 lain 的 layer1 应用和刚刚部署的几个 demo 应用.

```
>>> Available repos are :
ipaddr-resource   tinydns   console   webrouter   lvault   ipaddr-service   ipaddr-client   resource.ipaddr-resource.ipaddr-client   registry
>>> Available apps are :
Appname                         AppType               MetaVersion                                                   State
registry                        app                   1462444675-eaaf81179d90347dbd1e9b75a5469a63f8459021           healthy
console                         app                   1462792389-1b8fb1fc8e89c0d1ea9202a88d3955d545f8d1f6           healthy
webrouter                       app                   1462260224-41bff528f481a2ec6b01ca975fd17dea1b67bdb5           healthy
lvault                          app                   1462443822-81ed22691e022c788be3cb1e5cc1259c54daafcd           healthy
ipaddr-service                  service               1462790282-60f77d22799d8823ef771faef97897d60ca9c4b1           healthy
ipaddr-client                   app                   1462784181-9948592a0612d2ff551eb87e42bde6e091c47584           healthy
resource.ipaddr-resource.ipaddr-client  resource-instance     1462784153-944220ca13e9aae08412875990686e18b71bff9e           healthy
tinydns                         app                   1462424821-15ab41349dce81ddb9bc5c10309bb43b9aa0c573           healthy
ipaddr-resource                 resource              1462784153-944220ca13e9aae08412875990686e18b71bff9e           healthy
```

可以按照如下方式，进一步测试部署是否成功和集群的功能是否正常.

```
curl ipaddr-client.ipaddr-resource.resource.lain.local  # 期望返回 resource calico ip
curl ipaddr-service.lain.local  # 期望返回  service calico ip
lain scale -t resource.ipaddr-resource.ipaddr-client -n 2 local web  # 给 resource instance 扩容
```

这时，我们将 client, service, resource 都扩容了，
可以 `curl ipaddr-client.lain.local`, 多次请求既可以看到 resource / service / client 的 ip 轮流变化的情况.

## 视频演示

本视频通过上述应用展示了如何通过lain-cli实现应用的构建、部署、升级、测试和管理。

视频地址：http://www.bilibili.com/video/av4673273/

> 完整的lain-cli说明文档请见LAIN-白皮书3.3节：[lain-cli使用手册](../usermanual/sdkandcli.html)。
