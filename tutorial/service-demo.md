# [Service](../usermanual/service.html) 示例

对于[第一个 LAIN 应用](first-lain-app.html)中提到的 `hello-world` 来说，
假如想统计访问次数，我们可以把次数信息存在 redis 里。LAIN 提供了 [service](../usermanual/service.html)
机制来支持这种场景。

## 前置条件

- 首先需要一个 LAIN 集群。如果没有的话可以参考[安装 LAIN 集群](../install/cluster.html)启动一个本地集群，建议由 2 个节点组成
- 其次需要本地的开发环境。具体步骤见[安装 LAIN 客户端](../install/lain-client.html)
- 还需要[第一个 LAIN 应用](first-lain-app.html)中提到的 `hello-world`

## 部署 [redis-service-sm](../outofbox/redis-service-sm.html)

LAIN 提供了开箱即用的 [redis-service-sm](../outofbox/redis-service-sm.html)，按照以下步骤即可部署到 LAIN
集群：

```
git clone https://github.com/laincloud/redis-service-sm.git
cd redis-service-sm/
lain build
lain tag local
lain push local
lain deploy local
```

## 修改 lain.yaml

现在我们需要在 lain.yaml 里配置 service：

```
appname: hello-world

build:
  base: golang:1.8
  script:
    - go get -u github.com/go-redis/redis  # 安装依赖
    - go build -o hello-world
    
use_services:
  redis-service-sm:  # 提供 service 的应用的 appname
    - redis  # 提供 service 的应用里的 proc 的名字，这个 proc 需要定义相应的 portal

proc.web:
  type: web
  cmd: /lain/app/hello-world
  port: 8080
```

> - redis-service-sm 是服务提供者的 appname，如
>   [laincloud/redis-service-sm/lain.yaml](https://github.com/laincloud/redis-service-sm/blob/master/lain.yaml)
>   中的 appname 所示
> - redis 是 redis-service-sm 里提供服务的 proc，需要定义相应的 `portal.portal-redis`
>   如 [laincloud/redis-service-sm/lain.yaml](https://github.com/laincloud/redis-service-sm/blob/master/lain.yaml)
>   中的 proc.redis 和 portal.portal-redis 所示

## 修改 main.go

```
package main

import (
	"fmt"
	"net/http"
	"strconv"

	"github.com/go-redis/redis"
)

const visitCountKey = "visitCountKey"

func main() {
	redisClient := redis.NewClient(&redis.Options{
		Addr: "redis:6379",
	})

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		if err := redisClient.Incr(visitCountKey).Err(); err != nil {
			fmt.Fprintf(w, "redisClient.Incr() failed, error: %s.", err)
			return
		}

		visitCountStr, err := redisClient.Get(visitCountKey).Result()
		if err != nil {
			fmt.Fprintf(w, "redisClient.Get() failed, error: %s.", err)
			return
		}

		visitCount, err := strconv.Atoi(visitCountStr)
		if err != nil {
			fmt.Fprintf(w, "strconv.Atoi() failed, error: %s.", err)
			return
		}

		fmt.Fprintf(w, "Hello, LAIN. You are the %dth visitor.", visitCount)
	})

	http.ListenAndServe(":8080", nil)
}
```

`hello-world` 使用了 `redis:6379` 访问 `redis-service-sm` 提供的 `redis` 服务，其中：
- `redis` 是提供服务的 proc 的名字，如
  [laincloud/redis-service-sm/lain.yaml](https://github.com/laincloud/redis-service-sm/blob/master/lain.yaml)
  中的 proc.redis 所示，这个域名是由 LAIN 解析的
- `6379` 是提供服务的 proc 对应的 portal 监听的端口，如
  [laincloud/redis-service-sm/lain.yaml](https://github.com/laincloud/redis-service-sm/blob/master/lain.yaml)
  中的 portal.portal-redis.port 所示

[laincloud/hello-world@v2.0.0](https://github.com/laincloud/hello-world/tree/v2.0.0) 的完整代码在这里。

## 验证

`lain build && lain tag local && lain push local && lain deploy local`

然后访问 `hello-world`：

```
[vagrant@lain hello-world]$ curl http://hello-world.lain.local
Hello, LAIN. You are the 11th visitor.
```

如果看到了 `Hello, LAIN. You are the ${n}th visitor.`，说明我们的应用已经成功的访问了
`redis-service-sm` 提供的 `redis` 服务。
