# 在阿里云上安装 Lain

## 机器选型

- 网络类型使用*专有网络*
  - 专有网络创建:

    TODO

- 安全组创建

  TODO

- 修改 hostname。如 `hostname lain-01`(重新登录)

## Bootstrap

```sh
# @lain-01
git clone git@github.com/laincloud/lain.git lain
cd lain
# DOMAIN 默认值是 lain.local
./bootstrap -r registry.aliyuncs.com/laincloud --ipip --domain DOMAIN
```

## Add node (optional)

```sh
# @lain-01
lainctl node add -p playbooks -q lain-02:NODEIP
```

## Scale webrouter

参见 [webrouter scale](../maintain/webrouter.html#scale)

webrouter 扩容后，集群有两个外界入口(80 和 443 端口)，可使用阿里云的 ELB 引入外部流量。并将自己的域名解析到 ELB 上。
