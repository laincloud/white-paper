# SSO 运维文档

sso 支持 HA，由于每个 sso 的 container 是无状态的，
唯一的 proc 为 web 类型，所以如果 container 挂掉，只需要重启即可。 

## sso 的依赖

sso 依赖一个 mysql 数据库，程序会自动创建 schema, 所以只需要一个空库；
另外，还需要一个 smtp 服务。虽然 SMTP 是可选的，但是若没有 SMTP，
每一个新注册的用户都需要 DBA 帮忙才能激活用户。

### sso 启动时的一些重要参数

- domain: 指接受的注册用户的邮箱后缀, 为空或者使用默认值表示不进行检查。
- from: 激活、修改密码邮件等 sso 发送的邮件的发件人
- mysql: 存储所用的 mysql 数据库的 DSN
- site: sso 的网址
- smtp: sso 依赖的 smtp 的地址
- web: sso 后台服务的端口
- sentry: LAIN 的 sentry 的 DSN，可选
- debug: 指是否输出 debug 信息

sso 启动时，可以利用 LAIN 的秘密文件配置方法配置敏感信息，
然后在 run.sh 中利用 source 命令得到相关变量的具体值。 

注: lvault 服务器端不依赖 sso,  
可以利用 lvault 将 sso 的配置写入。
而 lvault 的 web 页面依赖 sso，在当前版本下，
我们推荐用命令行方式配置 secret files.

在 sso 部署之前，需要创建一个空的 mysql 数据库，然后将其 DSN 作为启动参数传入。
如果要用已有的 sso 的数据库，即 sso 涉及域名变化时，由于 sso 本身作为自己的一个 client, 所以需要手动去数据库里面更改 app 表中相关 sso 的几个 app 的 redirect_uri, 主要包括
SSO，SSO-Site，以及可能的关于 swagger-ui client.

#### 下面给出一个 sso 部署的样例

需要首先下面几个前置条件
1. [初始化并解锁 lvault](lvault.html#初始化和解锁).
1. 新建一个 mysql 数据库，目前版本的 sso 没有使用 mysql-service.

然后进入 lain-box, 执行
```
git clone https://github.com/laincloud/sso
cd sso
lain reposit local
lain build
lain tag local
lain push local
lain secret add local web /lain/app/secrets 'MYSQL="sso:password@tcp(rm-2ze.mysql.rds.aliyuncs.com:3306)/sso"'
lain deploy local
```

在 lain secret add 时，也可以直接指定文件，见 lain [命令行](../usermanual/sdkandcli.md#secret), 
同时也可以指定更多的启动参数，建议配置 smtp 服务。
一个 secrets 文件样例如下：

```
EMAIL="example.com"
DEBUG="false"
SMTP="mail.example.com:25"
MYSQL="sso:password@tcp(127.0.0.1:3307)/sso"
SENTRY="http://97:06@sentry.lain.local/3"
```

注意到 [run.sh](https://github.com/laincloud/sso/blob/master/run.sh) 中并没有用到所有的启动参数，
所以可以修改这个文件，加上一些其它参数，以满足进一步的需求，比如对 admin 邮箱和密码的初始化。

然后，即可测试：修改本地 /etc/hosts, 打开浏览器访问 https://sso.lain.local

### sso 的 swagger-ui 的 auth server 配置
sso 网站上的 API 文档的链接指向一个 swagger-ui，
这个 swagger-ui 的使用需要一个 oauth2 的认证，即 swagger-ui 本身作为 sso 的一个 client，sso 的管理员根据自己 sso 的域名，需要修改如下项的默认值.

```
sso/apidoc/swagger.json:10:    "host": "sso.example.com",
sso/apidoc/swagger.json:513:            "authorizationUrl": "https://sso.example.com/oauth2/auth",
sso/apidoc/swagger.json:514:            "tokenUrl": "https://sso.example.com/oauth2/token",
```

并且，为 swagger-ui 注册应用，并修改 apidoc/index.html 中的 appName，clientId，clientSecret 等值.

## sso 的用户（End User）注册

sso 的用户注册需要邮箱激活。问题是很多同学收不到邮件。
那么这时候可能会需要 SA 帮忙激活一下。步骤如下：

- 首先搞清楚注册哪个集群，找 DBA 去查 `user_register` 表, 查到正在注册用户的 `activation_code`.
- GET `https://{LAIN_DOMAIN}/api/activateuser?code={activation_code}`

## sso 的管理员

这里简要地介绍一下 sso 的管理员的相关问题。

sso 的管理员即 sso 中，在 admins 组里的成员。所以，sso 的管理员有两个可能的身份，
admin/normal, admin 身份即可以管理 admins 组.

sso 的管理员权限：

1. 可以查看所有的 sso 用户，删除用户（同时会从直接包含该用户的所有组中删除该用户），然而这些操作是否成功取决于用户接口的实现，比如当前的 sso-mysql 是可以的，而 sso-ldap 的用户管理不在 sso 这边，在 ldap 的管理者那里。 
1. 然而当前不支持删除组，即 sso 的管理员并没有其它组的管理权限，任何组的管理权限均只在该组的 admin 成员手中。

