# LAIN Console 管理

## 1. Console auth 使用

>在 console 中使用 auth 的方法

console 目前仅支持 `lain sso` 组件作为后端 auth server，因此必须确认 console 所在集群环境已经部署且能够访问到 auth server 

此后，需要对 etcd 的值进行修改以开启 auth：

在 etcd 中 设置 
```bash
etcdctl set /lain/config/auth/console '{"type": "lain-sso", "url": "https://sso.domain"}'
```

**注意：** 此后可通过 `etcdctl rm /lain/config/auth/console` 将 auth 关闭

此外，如果 console 使用了 archon 并且想要打开 auth，可以采用如下步骤：

- 在 sso 中注册一个新的 client，设置回调 url 为 `http://console.<domain>/api/v1/authorize/` 

- 为 lain.yaml 中所有 proc 配置 secret_files 为 config 文件

- 使用 lvault 添加 secret file，主要包括 client_id，client_secret 以及 redirect_uri 选项，文件内容参考代码库中的 config 文件


### registry auth

console 中同样提供对 registry 开启 auth 的支持，开启 registry auth 的过程中，需要访问 sso 验证用户的用户名与密码，因此需要提供 sso 的 client id 及 redirect uri。

console 的默认配置可以正常运行，如果想更新这些配置，可以对 entry.sh 文件进行配置，具体步骤如下：

- 在 `lain sso` 中申请 client id 及 client secret，回调 url 可以设置为： `scheme`://console.`domain`api/v1/authorize/

- 确保 entry.sh 中存在相应域名对应的 client id 及 client secret，没有则进行添加

- 重启 console 容器


## 2. Console 与 Sentry

Console 中可以支持 Sentry，只需在 console/settting 文件中将 `RAVEN_CONFIG` 替换为注册的 sentry dsn 即可开启 Sentry。此后，当应用部署失败时会自动往 sentry 中发送消息。


## 3. Console log

Console 中包含四种 log 文件，均存在 logs 目录下：

- access.log：gunicorn 的 access log，记录了访问 console 的信息

- error.log：gunicorn 的 error log

- console_log：记录了普通的 console 运行 log，可以在 `entry.sh` 文件中调整 log level

- console_op：记录了用户在 console 中部署 app 的操作，包括操作类型、操作人（开启 auth 情况下）、升级版本号等信息
