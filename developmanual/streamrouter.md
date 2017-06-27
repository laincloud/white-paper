# Streamrouter

## Introduction

Streamrouter 是 LAIN 的传输层代理，基本架构类似于 Webrouter，也是 watcher + nginx 的模式。

Streamrouter 的特性：
- 及时更新 Upstream：Streamrouter 能够及时根据应用的 proc 中配置的 ports 更新 upstream 配置文件。
- 支持主备：当部署的 Streamrouter 的实例个数大于1时，LAIN 的 networkd 组件会自动监听所有的 Streamrouter 实例的工作状态，并将Streamrouter 的虚拟IP绑定到正常工作的 Streamrouter 实例所在的节点上。如果该实例出现问题，networkd 也会及时地将该虚拟IP迁移到正常的容器所在的节点上。Streamrouter 各个实例之间则不会相互影响。

> Streamrouter 依赖的组件：
> - lainlet（必需）
> - networkd（必需）
> - rebellion （可选，日志收集）
> - kafka （可选，日志收集查询）
> - graphite (可选，配置文件语法合法性监控)

## Extension

目前 Streamrouter 依赖了 Nginx 的 stream 模块以及第三方开发的 stream_healthcheck 模块。如果之后有更好的解决方案，则可以相当方便地通过扩展 watcher 程序，实现代理组件的替换。

扩展方式通过实现 backend 包的 `Backend` 接口即可（可以参考 `HaproxyBackend` 和 `NginxBackend`），接口说明如下：

```go
type Backend interface {
    // Check is to check whether the configuration files are valid
	Check() error
	// Reload forces the program reloading its configuration files
	Reload()
	// RenderStreamFiles rendering corresponding configuration files according the app configurations
	RenderStreamFiles([]model.StreamApp) error
}
```

实现后，修改 `dispatcher/dispatcher.go` 中的 dispatcher 初始化代码即可。

```go
dispatcher := Dispatcher{
		notify:  make(chan interface{}),
		backend: &backend.NginxBackend{},
}
```

> 强烈建议对新的 Backend 实现做好相应的单元测试