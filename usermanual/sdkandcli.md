# CLI

CLI 是用户使用Lain平台的入口工具。用户对自己应用的部署，升级，查看等操作都要通过 CLI。

## 安装

### 安装SDK

```sh
git clone git@github.com:laincloud/lain-sdk.git
cd lain-sdk
python setup.py install
```

### 安装CLI

```sh
git clone git@github.com:laincloud/lain-cli.git
cd lain-cli
python setup.py install

```

<!-- lain app command -->

## 命令参考

CLI 的诸多命令中，大多数都会有一个 PHASE 参数，PHASE 是一个集群的标识，表示这个命令要在哪个集群上执行。如 `lain deploy local` 表示要把应用部署到名为`local`的集群上。
对于 PHASE 的设置，请参考 [config](#config) 命令。下面的参数说明中会忽略掉 PHASE。

以下各个命令，都可使用 `lain -h` 查看到。

### appversion

`lain appversion PHASE`

查看应用**在集群中**所有的可用版本。

### attach

`lain attach PHASE [-t TARGET] PROC INSTANCE_NUMBER`

- `-t/--target`: 指定要跟踪的应用，默认使用当前目录所属的应用。

跟踪集群中的该容器的标准/错误输出，`<Ctrl>-C` 退出。该功能依赖于 **Entry** 应用，默认是不能使用的，只有当集群中部署 **Entry** 才能使用。

关于 **Entry** 的更多信息，请见 [Entry 应用的相关文档](https://laincloud.gitbooks.io/white-paper/content/outofbox/entry.html)。

### backup

`lain backup SUB_COMMAND ...`

备份命令。请参考 [backup命令手册](backup.html#lain-backup命令手册)。

### build

`lain build [--push] [--release]`

- `--push`: 自动打tag并push到集群中。相当于 `lain build && lain tag PHASE && lain push PHASE`，而 PHASE 的值是全局配置 `private_docker_registry` 的值，可使用 `lain config show` 查看。默认关闭。
- `--release`: 如果本地已经存在 `{appname}:build` image，则以此镜像为基础构建 release image。

### check

`lain check PHASE`

检查该应用在集群中是否存在最新版本的 release 以及 meta image。

### clear

`lain clear --without WITHOUT`

- `--without`: 不清理哪些 tag，可指定多个，用 `,` 隔开，如 `--without build,meta` 表示不清理 meta 和 build 。

构建和测试应用时，会产生多个 image，其中包括构建中间态和最终结果。如 `prepare`, `release`, `meta`, `build`, `test`。该命令用来清除这些 image。

`lain rmi` 清理的是那些打了版本号的用于上线 image，而 `lain clear` 清理的是这些没有版本号的原始 image。

### config

`lain config SUB_COMMAND ...`

CLI 的配置管理

- **show**

  `lain config show`

  查看 CLI 的配置信息。

- **save**

  `lain config save PHASE KEY VALUE`

  修改某个 PHASE 的配置。如果 PHASE 不存在，自动创建。

  一个集群的最基本的配置就是它使用的域名，以 `lain.local` 为例，命令如下:

  ```sh
  lain config save local domain lain.local
  ```

  若集群开启了 SSO 认证，则需要设置 `sso_url`，然后才能执行 `lain login` 和 `lain logout`。如:

  ```sh
  lain config save local sso_url https://sso.lain.local
  ```

- **save-global**

  `lain config save-global KEY VALUE`

  修改全局设置。如设置 `private_docker_registry` 的值。这个地址 `lain build --push` 和 prepare 等功能都要依赖的。例如:

  ```sh
  lain config save-global private_docker_registry registry.lain.local
  ```

### dashboard

`lain dashboard [-s {appname, apptype, metaversion, state}] PHASE`

- `-s/--sort`, 指定列表的排序方式。
   - `appname`: 按应用名称排序，默认。
   - `apptype`: 按应用类型排序。
   - `metaversion`: 按应用的版本排序。
   - `state`: 按应用的状态排序。

显示集群中可部署的应用列表，以及已部署的应用的名称、类型、版本和状态。

### debug

`lain debug PROC`

`lain run` 以后，本机上会运行一个容器，`lain debug` 可进入到容器内部环境。

> `lain.yaml` 配置的 release.dest_base 中必须支持 `bash` 命令，否则该命令会失败。

### deploy

`lain deploy [-v VERSION] [-t TARGET] [-p PROC] [-o {pretty,json,json-pretty}] PHASE`

- `-v/--version`: 指定要部署的版本，默认使用最新版本。
- `-t/--target`: 指定要部署的应用，默认使用当前目录所属的 app
- `-p/--proc`: 指定要部署的 proc ，即只部署应用的某个 proc 。不能与 `-v/--version` 一起使用。
- `-o/--output`: 指定该命令的输出格式，默认 pretty 。

部署应用，即上线操作。如果应用在集群中已部署过，则该操作是一个升级操作。该命令执行后，会等待应用部署成功。若长时间无法部署成功，可 `<Ctrl>-C` 退出。然后通过 `lain ps` 等命令查看应用的状态。

### enter

`lain enter [-t TARGET] PHASE PROC INSTANCE_NUMBER`

直接进入集群环境中的某个 proc 的某个容器实例内部。该功能依赖于 **Entry** 应用，默认是不能使用的，只有当集群中部署 **Entry** 才能使用。

本质上是一个 `docker exec ...` 命令。与 `lain debug` 不同，`enter`是直接进入集群中某个容器内部，而 `lain debug` 是进入本地的容器。

关于 **Entry** 的更多信息，请见 [Entry 应用的相关文档](https://laincloud.gitbooks.io/white-paper/content/outofbox/entry.html)。


### login

`lain login [-c CID] [-s SECRET] [-r REDIRECT_URI] PHASE`

- `-c/--cid`:
- `-s/--secret`:
- `-r/--redirect-uri`:

`lain login` 时，默认会使用 sso 为 lain-cli 初始化的一组 client id 和 client secret。
如果指定，需要指定 sso 中已注册的应用的 client id, client secret 以及 redirect URI。更多相关知识可以参考 [sso 用户手册](sso/sso.html#应用注册)。

`lain login` 即登录指定的 sso，得到 access_token, access_token 会过期，过期后需要重新登录或者 [refresh](#refresh)。

### logout

`lain logout PHASE`

退出登录。

### meta

`lain meta`

查看应用**在本地**的最新版本号。

### prepare

`lain prepare`

为了加速编译构建 prepare image，将依赖一次性装好，
可以在 `lain.yaml` 中 build 下添加 prepare，
规定版本号和脚本.

### prepare-update

`lain prepare-update`

在已有的 prepare 镜像的基础上，更新 prepare 镜像。

### ps

`lain ps [-o {pretty,json,json-pretty}] PHASE`

- `-o/--output`: 指定该命令的输出格式，默认 pretty 。

显示应用当前的部署信息。

### push

`lain push PHASE`

将构建的镜像推送到集群。

build 出的镜像只存在于本地机器，需要 push 到集群中，才可以被部署。

### refresh

`lain refresh PHASE`

用于刷新本地的 access token。

由于 login 本质是得到 access token，
而这个 access token 会过期。一般来说，过期后可以重新登录；
然而，根据 OAuth2 协议，也可以通过 refresh token 得到新的
access token，避免重新登录。

### reposit

`lain reposit PHASE`

在集群中注册一个应用，是有一个被注册过的应用才可以被Push和部署。

### rm

`lain rm PROC`

删除被 `lain run` 起来的容器。

### rmi

`lain rmi PHASE`

删除本机上属于该应用的所有镜像。

每次的 `lain tag` 操作都会产生新的镜像。在一台机器上多次构建应用后，会积累很多历史镜像，因此可通过该命令清理。

### run

`lain run PROC`

在本地直接根据该 proc 的定义启动一个容器。开发者可使用这个命令在本地对应用进行手动测试和观察。

### scale

`lain scale [-t TARGET] [-c CPU] [-m MEMORY] [-n NUMINSTANCES] [-o {pretty,json,json-pretty}] PHASE PROC`

- `c/--cpu`: 每个实例可使用的CPU核数。
- `-m/--memory`: 每个实例分配的内存数。
- `-n/--numinstances`: proc 的运行实例数量。
- `-t/--target`: 指定要部署的应用，默认使用当前目录所属的 app。
- `-o/--output`: 指定该命令的输出格式，默认 pretty 。

Scale 操作，修改可使用的资源上限。更多详细的信息可参考 [scale](scale.html)。

### secret

`lain secret {show,add,delete} ...`

`lain secret show PHASE procname`

显示当前应用对应 proc 的所有配置。

- procname 为在 lain.yaml 中定义的 proc 的名字，procname 在 app 中是唯一的。

`lain secret add [-h] [-c CONTENT] [-f FILE] phase procname path`

为当前应用的对应 proc 增加路径为 path，内容为 content 的秘密配置文件.

- path 为秘密配置文件绝对路径，即已 `/` 开头的路径。
- content 可以通过 `-c` 直接指定，也可以使用 `-f` 指定某个文件。

`lain secret delete PHASE procname path`

删除当前应用和 proc 的对应路径下的秘密配置文件。

### stop

`lain stop PROC`

停止被 `lain run` 起来的本地容器。

### tag

`lain tag PHASE`

为构建出的镜像打标签，即版本。

> 版本号是根据应用 git 仓库的最新 commit id 生成的。所以如果只是改了代码，而没有 commit，那么 build 出来的结果虽是最新的，但是 tag 的版本号不会变。

### test

`lain test`

执行 `lain.yaml` 中定义的测试命令，请参考 [Tour](tour.html)。

`lain test` 根据 `lain.yaml` 的配置，先对应用进行构建并测试。


### undeploy

`lain undeploy [-t TARGET] [-p PROC] PHASE`

- `-t/--target`: 指定要反部署的应用，默认使用当前目录所属的 app。
- `-p/--proc`: 指定要反部署的 proc ，即只反部署应用的某个 proc。默认会把所有的 proc 都反部署。

反部署应用，即下线操作。

### update

`lain update`

更新 CLI 和 SDK 版本。

### validate

`lain validate`

检查应用配置是否合法，即该应用的`lain.yaml`的正确性。

> 当使用 resource 机制时，lain validate 的检查可能并不确切，可以根据情况忽略掉。

### version

`lain version`

查看 CLI 和 SDK 的版本。
