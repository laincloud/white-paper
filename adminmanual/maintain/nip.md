# Node IP

需要对外暴露服务的 app，需要设置 NIP。集群外可以通过 NIP 访问对应 app 的服务。Networkd 会在存在 app proc 的 node 设置好对应的转发规则。

## 配置 IP

在 etcd 的 `/lain/config/vips/` 创建一个名为 `0.0.0.0:PORT` 键，比如 `0.0.0.0:80` ，其值为一个 JSON 对象（为了方便阅读加了空格和缩进）：

```json
{
    "app": "resource.elb.webrouter",
    "proc": "haproxy",
    "ports": [
        {
            "dest": "80",
            "proto": "tcp"
        }
    ]
}
```

以下是这个键值的 [json schema](http://json-schema.org) :

```json
{
    "title": "Lain Virtual IP Config Schema",
    "type": "object",
    "properties": {
        "app": {
            "type": "string"
        },
        "proc": {
            "type": "string"
        },
        "ports": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "proto": {
                        "enum": [
                            "tcp",
                            "udp"
                        ]
                    },
                    "dest": {
                        "type": "string"
                    }
                },
                "required": ["proto"]
            },
            "minItems": 1,
            "uniqueItems": true
        },
        "excluded_nodes": {
            "type": "array",
            "items": {
                "type": "string"
            }
        }
    },
    "required": ["app", "proc", "ports"]
}

```

## 维护

配置好的 NIP 可以在 etcd 查到对应的配置, 例如: `/lain/networkd/apps/webrouter/worker/192.168.77.21:80`

## 应用配置

当在 etcd 中配置之后，各个节点上的 networkd 会得到通知，如果指定的 proc 部署在了本节点上，则 networkd 会激活节点配置 iptables 规则。

当指定的 proc 部署发生变化时，相关的 networkd 也会作出正确的配置动作。
