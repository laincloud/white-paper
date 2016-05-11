
#认证/授权

LAIN 上的认证（authentication）和授权（authorization） 
构成了 LAIN 安全的重要组成部分，
我们将其统称为 auth，lain 的 auth 涉及了 lain 上的很多组件。

lain 的认证／授权涉及了 sso，console，lainctl，lain-cli 等 lain 的几个组件，
这个文档介绍了如何开启 auth，和如何加强 auth 的安全性。

## 打开 auth

### Precondition

LAIN 中所有的关于 auth 的最终认证后端都为 sso，因此首先需要先部署好 sso，具体部署方法见 sso 文档。

在 sso 部署后会初始化两个组：admins 与 lain，其中都包含一个 sso 初始用户 admin。
其中 admins 是 sso 的管理员组，lain 是 lain 上所有应用的管理员组。

LAIN 中的 lainctl，lain-cli，以及 console 都需要直接与 sso 进行通信，因此需要为每一个组件在 sso 中注册一个应用以得到相应的 client id 与 client secret，sso 初始时已经创建了一个默认的应用：`client id: 3，secret：lain-cli_admin，redirect-uri：https://example.com `，并将其默认设置在了各组件所需要的位置，管理员同样可以在 sso 中重新注册并将其进行替换。

开启 auth 后用户执行 lain-cli 相应命令时，需要先执行 `lain login` 操作进行认证；


### 开启 app 维护相关 auth 步骤

开启 app 维护相关 auth 之后，不会允许用户随意对 app 进行 deploy 及相关操作，开启的步骤如下所示：

1. 使用 `lainctl auth init` 命令为已经存在于集群中但未在 sso 中创建组的 app 创建组。在命令执行过程中，需要先登录，然后以登录者的名义在 sso 中创建组，登录者则成为了相关 app 的 admin；

1. 在 sso 中将集群所有管理员添加进上文所提到的 lain 组中，然后将 lain 组作为子组依次添加进相应 app 对应的 sso 组中；

1. 执行 `lainctl auth open -s console` 命令开启 console 的 auth 选项，在使用命令时，需要指定 `type 及 sso url` 参数，


### 开启 image 维护相关 auth 步骤

开启 image 维护相关 auth 之后（registry auth），同样不会允许用户随意对相关 image 进行 push、pull 操作，能够进行操作的用户必须是该 app 的 maintainer。

registry 的 auth server 是 console，开启 registry 相关 auth 的步骤为：

1. 在 etcd 中设置 registry 访问白名单： `etcdctl set /lain/config/registry_ip_whitelist '192.168.77.21,192.168.77.22,...'` ，其中 IP 地址一般设置为集群节点的 IP 地址，或者其它某些有需要的节点的 IP 地址；

1. 由于白名单由 console 在初始化时读取，因此需要重启 console 容器以读取相关 IP 白名单；

1. 执行命令开启 registry auth： `lainctl auth open -s registry`，在使用命令时，需要指定 `realm, issuer, domain` 参数，具体含义可以参见 registry 维护文档

**注意:** 开启 registry 的 auth 之后，应该将 console scale 成两个实例，否则在升级 console 的过程中，可能因为 registry 找不到 auth server 而不允许拉去新的 console image。


## 增强安全性

为了使用户上手简单，我们将一些逻辑写到了代码里，
例如:
- sso 的初始管理员，用户名和密码均为 admin.
- sso 用默认的 client id 和 client secret.
- laincli 用默认的 client id 和 client secret.
- lain 组作为所有 console 创建的应用组的子组，在 lain 组中的成员拥有对所有其他应用的操作权限。

lain 的管理员可以通过调用 API，修改代码和修改 sso 数据库的方式改变默认行为。


