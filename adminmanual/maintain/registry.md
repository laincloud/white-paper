# LAIN Registry 管理

## Registry 配置

### 1. 配置方式

1. registry 的相关配置见官方文档: https://docs.docker.com/registry/configuration

2. 在 LAIN 中每次修改完 config.yaml 后需要重启 registry container 才能生效


### 2. storage 依赖配置
filesystem:
```
storage:
  filesystem:
    rootdirectory: /var/lib/registry
```

s3:
```
storage:
  s3:
    accesskey: ABCDEFGHIJ0123456789
    secretkey: AB12C3dE4fGhIvmMB4TZrpypR0rOJ2G5WhGUPn9L
    region: us-west-1 #important if use amazon filesystem
    regionendpoint: http://s3.domain.svc #optional endpoints 
    bucket: test
    secure: true
    v4auth: true
    chunksize: 5242880
    rootdirectory: /registry
```

### 3. auth管理
auth 相关信息见[文档](https://docs.docker.com/registry/spec/auth/)，LAIN 集群中如何打开 auth 可以参考 [auth 文档](../auth.md)。

#### 3.1 authorization server管理
LAIN 中的 token 规则采用 JWT(Json Web Token)，jwt 主要包括三部分 Header, claimset, signature，具体见[文档](https://docs.docker.com/registry/spec/auth/jwt/).

**注意：** 在authorization server 中最主要的是kid的生成，这部分在官网是没有对应文档的，需要根据docker source code来生成：https://github.com/docker/libtrust/blob/master/util.go#L158-L207


#### 3.2 auth配置

```
auth:
  token:
    realm: authorization server address
    issuer: "auth server"
    service: "domain"
    rootcertbundle: public certificate
```

LAIN 中 auth 的配置依赖 etcd，通过如下方式设置 registry 的 auth 信息:
`etcdctl set /lain/config/auth/registry '{"realm":"http://console.<domain>/api/v1/authorize/registry/", "issuer":"auth server", "service":"<domain>"}'`，其中 console 目前是 authorization server

### 4. 注意事项
```
如果config.yaml有问题，需要先docker rm registry，然后等待deploy拉起最新的registry
尽量先本地测试，保证config.yaml 无误后再重启registry，
storage 只能支持一种！
```

## Registry 操作

基本API, 请查阅: https://docs.docker.com/registry/spec/api/

### 1. 查看所有repo

```
curl -X GET /v2/_catalog
```

### 2. 查看repo下的所有tag

```
curl -X GET /v2/{repo}/tags/list
```

### 3. 清理image

1. 获取所有的repositories: curl -X GET /v2/_catalog

2. 通过repositories中获取对应repo的所有的tag信息: curl -X GET /v2/{repo}/tags/list

3. 软删除指定repo:tag的image: curl -X DELETE /v2/{repo}/manifests/{digses of tag}

4. registry节点的image blob文件删除: registry garbage-collect config.yaml


### 4. 注意事项

```
真正文件删除时最好不再对registry进行写操作
storage delete 必须配置为true
且为了兼容signature storage deprecating，需要如下配置，见：
https://docs.docker.com/registry/configuration/#compatibility
https://github.com/docker/distribution/issues/1661
```


## Registry 升级与扩容

- 当 registry 的存储后端为本地文件系统时，只能存在一个 instance，升级时需要在当前 registry 节点提前拉取最新的 registry image

- 当 registry 的存储后端为 moosefs 或 S3 之类的分布式文件系统时，可以存在多个 instance，但是当升级时如果只有一个实例则同样需要在存在 registry 的节点提前拉取最新的 registry image