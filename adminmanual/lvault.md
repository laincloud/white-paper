# lvault 维护文档

lvault 有两个重要的概念，分别是 key 和 root_token.
- key: 默认为 1 个，可以指定为 n(n>0) 个；若 n>1, 则由其中的一部分生成主密钥。key 的作用是解密存在 vault 存储后端的数据。
- root_token: ACL 控制手段。 

lvault 的一个基本原则是 key 和 root_token 只存在集群的内存中，若是发生了集群重启等重大事故，lvault 不能自己恢复，则需要人为解锁.

# API
以 /v1/ 为前缀会访问 vault 集群，只有需要更新密钥，锁某个节点等操作会用到。
以 /v2/ 为前缀会访问 lvault，一般来说，我们都利用 lvault 来方便处理 vault 的初始化，集群解锁等功能。

## /v2/init

### PUT

TODO：加入参数
当前默认的两个参数都是1，分别代表分发的 key 数目，和解锁至少需要的 key 的数目，感觉暂时没有需求去改变这两个参数 

对于一个集群只能执行一次，返回 key 和 root token，也是唯一的获得这两项的机会，对于系统管理员，一定要将其记在安全的地方。

**权限：**不需要认证

eg:
```
curl -X PUT http://lvault.lain.local/v2/init -d '{}'
curl -X PUT http://lvualt.lain.local/v2/init -d "{\"secret_shares\":1, \"secret_threshold\":1}"
```

## /v2/vaultstatus

### GET

查询 vault 集群的状态，返回值例如

```json
{
    "172.20.0.7:8200":{
        "Info":{
            "container_ip":"172.20.0.7",
            "container_port":8200
        },
        "Status":{
            "sealed":true,
            "t":1,
            "n":1,
            "progress":0
        }
    },
    "172.20.0.9:8200":{
        "Info":{
            "container_ip":"172.20.0.9",
            "container_port":8200
        },
        "Status":{
            "sealed":false,
            "t":1,
            "n":1,
            "progress":0
        }
    }
}
```

**权限：**不需要认证

## /v2/unsealall

### PUT

从 vault 集群只能对 instances 逐个解锁，这里为了方便，用该 api 将所有当前的 vault 实例解锁; 若 lvault 已经丢掉了 key 和 root_token，则必须先 reset.

eg:
```
curl -XPUT http://lvault.lain.local/v2/unsealall -d "{}"
```



**权限：**不需要认证

## /v2/reset

### PUT

```
curl -v -X PUT http://lvault.lain.local/v2/reset -d '{"keys":["193e7e1452679fcec7d06e96438752f4be7434abdc0d8632ac79c6cdb6911792"],"root_token":"4408bbd6-bc3f-1cba-08aa-6c0a12ff4e3d"}'
```

用传递过来的 key 来解锁，lvault 的 instances 记录 token. 为了安全，仅当该 lvault instance 的 token 丢掉或无效时，才会更新 token, 并且，还会对 token 进一步检查;
对于传递过来的 key，直接替换掉内存里的 key，这是由于这样可以保证 lvault 内存中的 key 已经失效，增强安全性。

reset 是为了当所有 lvault instaces 都同时重启，导致丢掉 root token 的情况。正常情况下，如果一个 lvault 节点重启， lvault 会在一段时间内自动更新该节点的 root token.


