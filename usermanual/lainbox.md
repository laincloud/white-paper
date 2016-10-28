# LAIN 应用的本地开发环境和管理环境

LAIN Box 项目： https://github.com/laincloud/lain-box

1. 提供基于虚拟机的本地应用开发环境
2. 提供对 LAIN 集群进行应用部署和管理的客户端环境

在 LAIN Box 中预装了 [LAIN CLI](https://github.com/laincloud/lain-cli)、[LAIN SDK](https://github.com/laincloud/lain-sdk)、Docker 以及常用的开发套件（Vim, Git）。开发者可在 LAIN Box 中本地编译、测试应用而不必在自己的电脑上安装各种依赖。LAIN Box 同样也可以用来管理 LAIN 集群。以管理 [LAIN Demo](https://github.com/laincloud/lain-demo) 集群为例，你可以修改 LAIN Box 的 `/etc/hosts` 文件，将 `.lain.local` 系列以域名指向 LAIN Demo 集群的虚拟 IP 地址，并用 `lain config` 命令配置好集群地址，即可进行管理。

LAIN Box 中默认会通过 Vagrant 将当前目录映射到 `/apps/lain/` 下。如果需要更多目录，可修改 `config.yaml` 文件，按需添加项目目录，然后通过 `vagrant reload` 重启 LAIN Box，即可在 LAIN Box 中访问到其他项目的目录。例子：

```yaml
apps:
    foo-project: ~/Projects/foo
    bar-project: ~/Projects/bar
```
> 请使用不高于1.7.4版本的vagrant，vagrant 1.8.x 版本会在虚拟机启动时因网络原因卡住。
