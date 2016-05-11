# Virtual IP

## 配置 IP

在 etcd 的 `/lain/config/vips/` 创建一个名为 `VIP` 键，比如 `192.168.77.254` ，其值为一个 JSON 对象（为了方便阅读加了空格和缩进）：

```json
{
    "app": "resource.elb.webrouter",
    "proc": "haproxy",
    "ports": [
        {
            "src": "80",
            "proto": "tcp"
        },
        {
            "src": "443",
            "proto": "tcp"
        },
        {
            "src": "5555",
            "proto": "udp",
            "dest": "53"
        }
    ]
}
```

目前只支持 master/backup 策略

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
                    "src": {
                        "type": "string"
                    },
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
                "required": ["src", "proto"]
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

配置好的 VIP 可以在 etcd 查到对应的 lock, 例如: /lain/networkd/vips/192.168.77.254.lock

## 应用配置

当在 etcd 中配置之后，各个节点上的 networkd 会得到通知，如果指定的 proc 部署在了本节点上，则 networkd 会激活节点配置 VIP 和 iptables 规则。

当指定的 proc 部署发生变化时，相关的 networkd 也会作出正确的配置动作。
