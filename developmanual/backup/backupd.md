# backupd

## 介绍

Backupd主要是为Lain集群上的app提供备份功能。使用者可在lain.yaml中配置volume的备份属性，对volume进行定时备份。

## 整体设计

分两层，controller和daemon。controller负责任务的获取和记录等，daemon负责任务的执行。对于用户而言，daemon是隐藏的组件，所有的备份相关请求都要走controller。

daemon以定时调度器为核心，即`cron-engine`。该模块具有定时调度的功能，并会把调度的结果post给指定的地址。engine所支持的任务类型，以插件的形式注册到engine上。

对volume的备份功就属于`cron-engine`的一个插件，我们通过api，把定时任务提交给`cron-engine`, engine就会根据设定在指定时间执行某插件。

controller 从lainlet中获取最新的配置并解析出备份任务，将任务发送给各个backupd-daemon。此外，还负责把一些查询的请求(如备份结果等)代理给某些daemon获取结果。

![design](img/backupd-design.png)


## 编译和运行

### 编译

**需要环境: go1.5+**
```sh
go build -o backupd
```

### 运行

**需要环境: lainlet, moosefs**

```sh
backupd --help 查看运行参数
```
