# LAIN 秘密文件配置系统使用方法

主要分为三步：

1. 用户注册某应用
2. 利用 lvault 的 API 或者图形界面，将 secret files 内容写入 lvault
3. 成功写入后，在 app 的 lain.yaml 中配置这些 secret files, 部署该 app

## Demo

本地集群的 app testsecret 需要用到 secret files

```
appname: testsecret # 全局唯一的应用名

build:  # 描述如何构建应用 image ，例如安装依赖等，供 release 和 test 使用
  base: golang  # 一个已存在的 docker image ，包含编译环境和默认的配置     
  script:  # 会在 docker container 的 /lain/app/ 下 clone 代码并执行下列命令（下同）
    - go build -o testsecret

release:
  dest_base: ubuntu:trusty
  copy:
    - src: testsecret
      dest: /usr/bin/testsecret

test:  # 描述如何从 build 的结果 image 执行测试
  script:   # 这些命令基于 build 的结果 image 运行，任何一个命令返回非0退出码即认为测试失败
    - go test
  
web:  # 定义一个 web 服务
  cmd: testsecret  # 基于 release 产生的发布用 image 运行
  port: 8013  # 服务监听的端口，默认为自动分配一个放在 `PORT` 环境变量里
  secret_files:
    - /a/b/hello
    - secret

notify:
  slack: "#hello"  # 应用相关的通知和报警都会发送到该 slack channel
```

上边的 lain.yaml 文件定义了两个 secret\_files，注意到相对路径实际上是相对于 '/lain/app' 下的路径，我们可以按如下方法将 secret\_files 的内容写入 lvault.
当前的 secret\_files 只支持 root 用户.

下面是用户拿到 access-token 后，可以直接调用 lvault 的 api 写入秘密文件：

```
curl -X PUT  -H "access-token:KIXoIm76QgeGt-ZhGWM11g"  "http://lvault.lain.local/v2/secrets?app=testsecret&proc=testsecret.web.web&path=/a/b/hello" -d '{"content":"hello yx"}'
curl -X PUT  -H "access-token:KIXoIm76QgeGt-ZhGWM11g"  "http://lvault.lain.local/v2/secrets?app=testsecret&proc=testsecret.web.web&path=/lain/app/secret" -d '{"content":"hello world"}'
```

另外，我们对这一过程做了简单的 UI 支持，请严格按照如下步骤进行：

* 打开浏览器访问 http://lvault.`{LAIN_DOMAIN}`/v2/
* 点击自助服务下的 “登录/ LVAULT 集群状态查询” 按钮，按照 app maintainer 的身份登陆；
* 回到首页，只能为已注册的 app 写入秘密文件；

之后我们只需要按照正常部署过程部署该应用，在应用的 container 里的相应路径下便会自动出现这两个文件.

## API

## /v2/secrets

### PUT
/secrets?app=hello&proc=proc1&path=a/b/c

body 为要写的 secret files，默认的格式为

```go
type Data struct {
	Content string `json:"content"`
	Owner   string `json:"owner,omitempty"`
	Group   string `json:"group,omitempty"`
	Mode    string `json:"mode,omitempty"`
}
```
所以，如果不能按照上述格式解析，则所有的 body 会被写入 content 字段。
owner 和 group 的默认值为 root，而 mode 的默认值为 400

返回：200

**权限：**需要认证，app maintainer 有权限访问

### DELETE
/secrets?app=hello&proc=proc1&path=a/b/c

**权限：**需要认证，app maintainer 有权限访问

### GET
/secrets?app=hello&proc=proc1
其中，app 是必选参数，proc 是可选参数.

返回被查询 proc 的所有 path 下的 secret data

**权限：**需要认证，app maintainer 和 sync 脚本有权限访问

返回：200

```json
[
    {
        "path":"zhiwang/makemoney/cy",
        "content":"100",
        "owner":"root",
        "group":"root",
        "mode":"400"
    },
    {
        "path":"zhiwang/makemoney/test",
        "content":"5",
        "owner":"zhiwang",
        "group":"bigdata",
        "mode":"400"
    }
]
```

## 一些例子

```
curl -X PUT  -H "access-token:KIXoIm76QgeGt-ZhGWM11g"  "http://lvault.lain.local/secrets?app=testsecret&proc=testsecret.web.web&path=a/b/hello" -d '{"content":"hello yx"}'

curl "http://lvault.lain.local/secrets?app=what&proc=who" -X GET -H "access-token:KIXoIm76QgeGt-ZhGWM11g"

curl "http://lvault.lain.local/secrets?app=what&proc=who&path=adfa" -X DELETE -H "access-token:KIXoIm76QgeGt-ZhGWM11g"
```


