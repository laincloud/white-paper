# Private Registry 简介

Private Registry是一个类似于docker hub的image仓库，用来存储自己的私有镜像，提高镜像的拉取效率。

在lain中，主要通过构建Private Registry用来存储应用镜像构建时产生的中间准备镜像，从而减少应用镜像的构建次数或者提高中间准备镜像的构建效率，从而提高lain build的效率。

# Private Registry 使用

```
1. Private Registry 主要作用于 lain cli.
2. 在构建lain应用镜像时需要首先配置private registry.
3. 通过如下方式配置 private registry：lain config save-global private_docker_registry '{private registry address}'
```

# Private Registry 原理

## lain build 简介

```
1. lain 应用发布时，主要分为lain build, lain tag, lain push, lain deploy 四个步骤
2. lain build 分为base, prepare, build 三个阶段
2. 其中base阶段拉取指定基础镜像
3. prepare阶段通过对base镜像进行操作生成{appname}-prepare-{timestamp}镜像
4. build阶段在prepare镜像基础上进行操作生成最终应用镜像
```
## private registry作用
```
1. private registry主要作用于lain build中的prepare阶段
2. 存储产生的prepare镜像
4. 当应用的prepare部分不动时，就可以跳过prepare阶段直接进行build
```