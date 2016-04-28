# console auth 使用文档

>在 console 中使用 auth 的方法

## 前提条件

console 目前仅支持 [lain sso](https://sso.yxapp.in) 作为后端 auth server，因此必须确认 console 所在集群环境能够访问到 auth server 

此后，需要对 entry.sh 文件 以及 etcd 进行修改以开启 auth

### entry.sh 文件

entry.sh 文件中默认提供了五个域名选项：lain.local, lain.bdp.cc, prod.bdp.cc, yxapp.in, yxapp.xyz。 如果 Lain 集群使用其中的某一个域名则不用修改 entry.sh 文件

如果需要使用新的域名则进行以下步骤：

- 在 [lain sso](https://sso.yxapp.in) 中申请 client id 及 client secret，回调 url 设置为： `scheme`://console.`domain`api/v1/authorize/

- 确保 entry.sh 中存在相应域名对应的 client id 及 client secret，没有则进行添加

- 重启 console 容器


### etcd

在 etcd 中 设置 
```bash
etcdctl set /lain/config/auth '{"type": "lain-sso", "url": "https://sso.yxapp.in"}'
```

**注意：** 此后可通过 `etcdctl rm /lain/config/auth` 将 auth 关闭

