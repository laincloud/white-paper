# 在阿里云上安装 Lain

## 环境配置

- 网络类型使用 **专有网络** ，然后在其中创建一个 **交换机**

- 其余步骤请参见 [快速安装](../../quickstart/install.html#云服务器)

## Scale webrouter

参见 [webrouter scale](../maintain/webrouter.html#scale)

webrouter 扩容后，集群有两个外界入口(80 和 443 端口)，可使用阿里云的 ELB 引入外部流量。并将自己的域名解析到 ELB 上。
