# LAIN 上应用构建部署示例

>在建立了 Lain 上的 App 和 Proc 概念后，
本节将一步步演示如何在 lain-box 利用 laincli 构建、部署应用.

## 准备
- 首先需要部署一个本地集群，建议由两个节点组成。具体步骤见[LAIN 的快速安装](install.html).
- 利用 lain-box 搭建本地的开发环境. 具体步骤见 [本地开发环境](../usermanual/tour.md#本地开发环境)

### 检查 lain-box 的设置
一般来说，在 lain-box 中是没有对 lain.local 域名劫持的，
所以需要修改 /etc/hosts.
```

# 加入 ( vip 模式 )
192.168.77.201  registry.lain.local console.lain.local entry.lain.local ipaddr-client.lain.local ipaddr-service.lain.local ipaddr-client.ipaddr-resource.resource.lain.local
# 或者 ( node 模式 )
192.168.77.21  registry.lain.local console.lain.local entry.lain.local ipaddr-client.lain.local ipaddr-service.lain.local ipaddr-client.ipaddr-resource.resource.lain.local
```

下面检查一下 lain config
```
lain config show
lain config save local domain lain.local
```



## 部署 demo 应用
这里，我们将部署三个示例应用，由于这三个应用有逻辑关系，建议按照下列顺序部署. 
更多背景知识可见 [service](../usermanual/service.html), [resource](../usermanual/resource.html).

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
lain dashboard local
lain reposit local
lain build
lain tag local
lain push local
lain deploy local
```

resource 是没有实例的, 所以不需要 lain ps local

### 部署 ipaddr-service

```
cd /home/vagrant/demo-ipaddr-service
lain dashboard local
lain reposit local
lain build
lain tag local
lain push local
lain deploy local
lain ps local
lain scale -n 2 local webc
lain ps local
```

这时候可以看到，ipaddr-service 的 proc list 已经有实例了，但是还没有 portal.

### 部署 ipaddr-client

```
cd /home/vagrant/demo-ipaddr-client
lain dashboard local
lain reposit local
lain build
lain tag local
lain push local
lain deploy local
lain ps local
lain scale -n 2 local web
lain ps local
```

这时可以看到一个叫做 resource.ipaddr-resource.ipaddr-client 的应用部署起来了.
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


