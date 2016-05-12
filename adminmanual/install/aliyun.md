
# 在阿里云上安装Lain

## bootstrap

```sh
# @lain-01

# 修改 hostname
hostname lain-01 # 修改后需退出重新登录ssh，才可看到效果。

git clone git@github.com/laincloud/lain.git lain

cd lain

# DOMAIN 默认值是 lain.local
./bootstrap -r registry.aliyuncs.com/laincloud --ipip --domain DOMAIN
```

## add node

```sh
# 同 lain-01 一样修改下 新节点的 hostname

# @lain-01
lainctl node add -p playbooks -q lain-02:NODEIP
```

## 导入外网流量

aliyun 控制台开启 elb 实例

随便华北 A或者B区开一个外网 负载均衡

配置端口监听以及后端服务器

## scale webrouter

参见 [webrouter scale](../maintain/webrouter.html)

webrouter 扩容后，集群有两个外界入口，可使用阿里云的 ELB 引入外部流量。并将自己的域名解析到 ELB 上。
