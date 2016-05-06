# Private Registry 简介

私有Registry是一个类似于docker hub的image仓库，用来存储自己的私有镜像，提高镜像的拉取效率。
Docker Registry是一个开源项目，遵循Apache license，[github地址](https://github.com/docker/distribution)

详细介绍见[官网](https://docs.docker.com/registry/)


# registry 基本使用

## 1. registry install

```
1. 获取registry src: git clone https://github.com/docker/distribution.git
2. 指定至具体版本: git checkout v2.4.0
3. 编译: make PREFIX=/go clean binaries (PREFIX 路径根据实际情况改动)
4. 在/go目录下会生成对应二进制文件: registry
```

## 2. Registry 配置
Registry 配置主要是针对日志，存储，auth，middleware，外界访问(http)，通知，缓存，代理及兼容性配置,并将相关配置写入 config.yaml。

详细配置信息见[官网](https://docs.docker.com/registry/configuration/)


## 3. registry 启动

```
1. 将registry移动至/usr/bin目录
2. registry serve config.yml
```

## <span id="use">4. registry 使用</span>

```
Assume Registry addr: localhost:5000 
1. build image: build -o myfirstimage . （Current work dir contains Dockerfile）
2. tag image: docker tag myfirstimage localhost:5000/myfirstimage
3. login(when auth required): docker login localhost:5000 (Enter Username, Password, Email(Optional))
4. push image: docker push localhost:5000/myfirstimage
5. pull image: docker pull localhost:5000/myfirstimage
```

# registry in lain


## 1. 运行方式

```
1. Lain Registry以Container的方式运行于lain中，由bootstrap 启动运行。
2. registry url 为: registry.{domin}，访问时需要修改Host至 node1 ip。
3. 默认的存储方式为filesystem，当然也可以支持S3，OSS等方式，需要自己配置。
4. 默认为no auth，可以通过lainctl 开启auth
5. auth规则由console控制
```

## 2. role in lain

```
1. registry 是lain layer1层应用，与laincli, console(layer1)及deployd(layer0)关联。
2. 用户部署lain应用时，lain cli会将相关应用的镜像push到lain registry中，并向console中注册应用信息。
3. deployd根据应用信息从lain registry pull镜像，并在lain中分配资源运行应用，也就是运行pull下来的镜像。
```

## 3. 使用

主要应用于应用lain cli发布，基本使用方式同[registry 使用](#use)。