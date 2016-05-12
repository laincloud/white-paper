# Private Registry 简介

Private Registry 是一个类似于docker hub 的 image 仓库，用来存储自己的私有镜像，提高镜像的拉取效率。

在 LAIN 中，主要通过构建 Private Registry 用来存储应用镜像构建时产生的中间准备镜像，从而减少应用镜像的构建次数或者提高中间准备镜像的构建效率，从而提高 lain build 的效率。

## Private Registry 使用

1. Private Registry 主要作用于 lain cli.

1. 在构建 LAIN 应用镜像时需要首先配置 private registry.

1. 通过如下方式配置 private registry：`lain config save-global private_docker_registry '{private registry address}'`


## Private Registry 原理

### lain build 简介

1. lain 应用发布时，主要分为lain build, lain tag, lain push, lain deploy 四个步骤

1. lain build 分为 base, prepare, build 三个阶段

1. 其中 base 阶段拉取指定基础镜像

1. prepare 阶段通过对 base 镜像进行操作生成 {appname}-prepare-{timestamp} 镜像

1. build 阶段在 prepare 镜像基础上进行操作生成最终应用镜像


### private registry 作用

private registry 主要作用于 lain build 中的 prepare 阶段，存储产生的 prepare 镜像

**注意:** 为方便起见，可以直接将 private regsitry 设置为当前集群的 registry 地址；如果设置成了其它开启 auth 的 registry，则需要首先使用 docker login 登录相应 registry。