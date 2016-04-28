# 配置和管理虚拟 IP

> # FIXME

## 创建 VIP

在 etcd 的 `/lain/config/vips/` 创建一个名为 `appname.procname` 键，比如 `webrouter.worker` ，其值为一个 JSON 对象（为了方便阅读加了空格和缩进）：

```json
{
    "vip": "192.168.77.254",
    "port": [80, 443],
    "loadbalance": "master-backup",
    "node_filters": ["edge==true"],
}
```

其中

`vip` 为需要绑定的 VIP;

`[80, 443]` 为 VIP 需要转发的端口；

`master/backup` 为负载均衡策略，也可以取值为 `round-robin`。

`node_filters` 为绑定该 VIP 的 proc 在部署时需要传给 deployd 的 filter 信息。

以下是这个键值的 [json schema](http://json-schema.org) :

```json
{
    "title": "Lain Virtual IP Config Schema",
    "type": "object",
    "properties": {
        "vip": {
            "type": "string"
        },
        "port": {
            "type": "array",
            "items": {
                "type": "integer",
                "minimum": 1,
                "maximum": 65535
            },
            "minItems": 1,
            "uniqueItems": true
        },
        "loadbalance": {
            "enum": [
                "master-backup",
                "round-robin"
            ]
        },
        "node_filters": {
            "type": "array",
            "items": {
                "type": "string"
            }
        }
    },
    "required": [
        "vip",
        "port",
        "loadbalance"
    ]
}
```

## 应用配置

当在 etcd 中配置之后，各个节点上的 lainlet 会得到通知，如果节点被 `node_filters` 选中，并且指定的 proc 部署在了本节点上，则 lainlet 会激活节点配置 VIP 和 iptables 规则。

当指定的 proc 部署发生变化时，相关的 lainlet 也会作出正确的配置动作。
