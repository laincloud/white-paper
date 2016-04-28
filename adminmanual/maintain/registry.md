# registry管理

## registry 配置

### 配置方式
```
1. registry的相关配置见官方文档: https://docs.docker.com/registry/configuration
2. 在lain中每次修改完config.yaml后需要重启registry container才能生效
```

### storage依赖配置

```
filesystem:

storage:
  filesystem:
    rootdirectory: /var/lib/registry


s3:

storage:
  s3:
    accesskey: ABCDEFGHIJ0123456789
    secretkey: AB12C3dE4fGhIvmMB4TZrpypR0rOJ2G5WhGUPn9L
    region: us-west-1 #important if use amazon filesystem
    regionendpoint: http://s3.bdp.svc #optional endpoints 
    bucket: test
    secure: false
    v4auth: true # use s3 signature v4 if true else signature v2
    chunksize: 5242880
    rootdirectory: /registry
```

### auth管理
auth 相关信息见[文档](https://docs.docker.com/registry/spec/auth/)

#### authorization server管理
lain中的token规则采用JWT(Json Web Token),jwt主要包括三部分 Header, claimset, signature，具体见: 见[文档](https://docs.docker.com/registry/spec/auth/jwt/).

```
注意：在authorization server 中最主要的是kid的生成，这部分在官网是没有对应文档的，需要根据docker source code来生成：https://github.com/docker/libtrust/blob/master/util.go#L158-L207
```

#### auth配置

```
auth:
  token:
    realm: authorization server address
    issuer: "auth server"
    service: "domain"
    rootcertbundle: public certificate
lain中auth的配置依赖etcd，通过如下方式设置registry的auth信息
etcdctl set /lain/config/auth/registry '{"realm":"http://console.<domain>/api/v1/authorize/registry/", "issuer":"auth server", "service":"<domain>"}'
其中console目前是authorization server
```

### 注意事项
```
如果config.yaml有问题，需要先docker rm registry，然后等待deploy拉起最新的registry
尽量先本地测试，保证config.yaml 无误后再重启registry，
storage 只能支持一种！
```

## registry 操作

```
基本API, 请查阅: https://docs.docker.com/registry/spec/api/
```
### 查看所有repo

```
curl -X GET /v2/_catalog
```

### 查看repo下的所有tag

```
curl -X GET /v2/{repo}/tags/list
```

### 清理image

```
1. 获取所有的repositories: curl -X GET /v2/_catalog
2. 通过repositories中获取对应repo的所有的tag信息: curl -X GET /v2/{repo}/tags/list
3. 软删除指定repo:tag的image: curl -X DELETE /v2/{repo}/manifests/{digses of tag}
4. registry节点的image blob文件删除: registry garbage-collect config.yaml
```

### 注意事项

```
真正文件删除时最好不再对registry进行写操作
storage delete 必须配置为true
且为了兼容signature storage deprecating，需要如下配置，见：
https://docs.docker.com/registry/configuration/#compatibility
https://github.com/docker/distribution/issues/1661
```