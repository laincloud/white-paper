# Concepts

## Lain cluster

一个 lain cluster 是由一次 bootstrap 建立起来的运行着多个 apps 及支撑 apps 运行的基础设施的集合。

Cluster 和 cluster 之间没有关系。

每个 cluster 会配置一个 cluster domain ，用作生成访问 app 需要使用的域名。例如 `lain.local` 。

[可选] 每个 cluster 会配置一个或多个外部虚 IP（virtual ip，或 vip），用作对 cluster 外访问提供入口。

## Node

一个集群内，每一个一个运行着 Linux 的服务器被称为一个节点(node）。该服务器可能是物理机，也可能是虚拟机。所有的节点之间可通过 IP 协议互通。

## App

每一个 lain app 有一个 cluster 唯一的 appname。它是一个或多个 proc 的集合。

对一个根目录有 lain.yaml 文件的代码仓库，按照 lain.yaml 中的配置进行 build 可得到一个 image 。对此 image 执行部署操作即可创建或者更新一个 app 。 

## proc

应用在运行时由一个或多个 proc instances 组成。proc 在 lain.yaml 中定义。定义方法详见 [Proc](proc.md) 。

每一个 proc 有一个应用内唯一的 proc name 。

## instance

app 会保证在集群中运行 `num_instances`（默认为1）个 `docker run {APP_IMAGE} {PROC_CMD}` 进程，每个被称为一个 proc instance ，并被授予 1 至 `num_instances` 中的一个唯一的数字为 instance number ，并在启动 instance 时作为环境变量 `LAIN_INSTANCE_NO` 的值。

应用管理员和集群管理员可以动态调整运行时的 `num_instances` ，在 app 更新时保留之前的 `num_instances` 不变，忽略 lain.yaml 中的变更。即，lain.yaml 中的 `num_instances` 配置只在新部署 app 时生效。（其他的运行时可变项也是类似处理。）

如果 proc 定义了 port ，则在**本app**的所有 proc instance 中，都可以直接使用 `{PROC_NAME}-{INSTANCE_NO}` 作为主机名，访问该 instance 的端口。

*app 应无法访问其他 app 的 instance*

## service provider

`portal` 类型的 proc ，用于对 cluster 内其它 app 提供服务。详见 [Service && Resource](service.md) 。

