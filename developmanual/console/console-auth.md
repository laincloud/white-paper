# console auth 使用文档

>在 console 中使用 auth 的方法

console 目前仅支持 `lain sso` 组件作为后端 auth server，因此必须确认 console 所在集群环境已经部署且能够访问到 auth server 

此后，需要对 etcd 的值进行修改以开启 auth：

## etcd

在 etcd 中 设置 
```bash
etcdctl set /lain/config/auth/console '{"type": "lain-sso", "url": "https://sso.domain"}'
```

**注意：** 此后可通过 `etcdctl rm /lain/config/auth/console` 将 auth 关闭


## registry auth

console 中同样提供对 registry 开启 auth 的支持，开启 registry auth 的过程中，需要访问 sso 验证用户的用户名与密码，因此需要提供 sso 的 client id 及 redirect uri。

console 的默认配置可以正常运行，如果想更新这些配置，可以对 entry.sh 文件进行配置，具体步骤如下：

- 在 `lain sso` 中申请 client id 及 client secret，回调 url 可以设置为： `scheme`://console.`domain`api/v1/authorize/

- 确保 entry.sh 中存在相应域名对应的 client id 及 client secret，没有则进行添加

- 重启 console 容器