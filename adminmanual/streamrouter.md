# Streamrouter 的维护

如果 LAIN 上有应用需要对外提供传输层的服务，如 TCP 服务；
或者希望为 LAIN 上其他应用提供服务，并且安全性要求不高的话，可以用 streamrouter 提供统一的代理。
默认可用端口为 [9500,10000] 共 5001 个端口, 可以在 console 的页面上看已经使用的端口。

## 部署依赖

1. 目前 streamrouter 还没有作为一个 layer1 的应用，并没有在 bootstrap 时创建 streamrouter 的 calico profile，所以，为了使 streamrouter 的流量可以转发到后端的容器，必须将起打上 lain 的 tag. 即：

`calicoctl profile streamrouter tag add lain`

1. LAIN 对外提供 stream 服务时，需要一组 VIP，例如 "10.100.170.213" "10.100.170.214", 需要管理员配置, 命令如下:

`etcdctl set /lain/config/streamrouter/vippool '["10.100.170.213","10.100.170.214"]'`

1. networkd 版本为 v0.1.17 及以后，networkd.service 需要添加 --streamrouter 启动参数.

## 问题解决

如果遇到 iptables 或者 calico profile 不能正常生成或者更新的情况，主要查看 etcd `/lain/config/vips/` 目录下是否生成了正确的 vip 配置，如果不是的话，请检查 lain-sdk, deployd, console 等组件是否没有达到规定的版本。
