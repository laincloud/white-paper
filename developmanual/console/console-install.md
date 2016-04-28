# console 部署文档

>在 Lain 的机制下对 console 进行编译部署

## Precondition

- 准备 lain.yaml 和 entry.sh 文件
- [Lain](http://docs.lain.bdp.cc) 的文档说明

## Usage

### 1. local dev/test env @ lain-in-vagrant

>`Target` : 基于 Lain 的机制提供一个方便的可重建的 docker 化的本地开发和测试环境
>`REF` : 建议阅读 [Docker Docs](https://docs.docker.com/)  [Docker —— 从入门到实践](http://yeasy.gitbooks.io/docker_practice/content/)  [Lain Docs](http://docs.lain.bdp.cc/)
>`lain-in-vagrant` 是基于 `vagrant` / `virtualbox` 的 一个虚拟机开发环境，其启动的虚拟机是一个集成了 `docker` 和 `laincli` 等工具的 `centos7.1` ，作为 `docker daemon` 的宿主系统

#### 1.1. lain 环境预备

- 宿主机（咱们一般都是 mac book 对吧）上代码目录（假设在 `/code/`） `cd /code/ && git clone http://laingit.bdp.cc/lain/lain-in-vagrant.git`
- 宿主机 `cd /code/lain-in-vagrant`
- 宿主机 `vim config.yaml` 按照其中的说明修改想要在宿主机和虚拟机之间映射的目录，建议将宿主机的 `console` 目录（假设在 `/code/console`）映射到虚拟机里的 `/apps/console`，即写成 `console: /code/console`
- 宿主机 `vagrant up` 启动虚拟机（第一次启动会下载一个几百M的系统模板，需要在办公网）
- 宿主机 `vagrant ssh` 登录虚拟机（默认用户 vagrant ，使用 `vagrant ssh` 时免密码，sudo 时也基本免密码，需要密码的时候 root/vagrant 用户的密码都是 vagrant ）
- 虚拟机 `cd /apps/console`
- 虚拟机 `lain prepare`  将远程 docker image 拉到本地并做指定的准备工作，看网络情况可能会花一些时间，后续考虑直接使用这些 base image 的厂内镜像

#### 1.2. local test

- 虚拟机/宿主机 do some coding && commit
- 虚拟机 `lain test`  在容器内执行本地测试，可以在 lain.yaml 中增加想要执行测试的指令

#### 1.3. local build 和 开发过程

>如出现任何问题，请联系 `lain 开发组`

- 虚拟机 `lain update`
- 虚拟机 `lain config save dev domain lain.bdp.cc`
- 虚拟机 `lain config save prod domain prod.bdp.cc`
- 虚拟机 `lain config save yxapp domain yxapp.in`
- 虚拟机/宿主机 do some coding && commit
- 虚拟机 `lain test`  构建镜像，并进行测试
- `git push`   代码提交推送
- `gitlab mergerequest`  代码审核
- `merge to master` 合入
- ci

#### 1.4. local debug server

>前提是先提供本地测试数据库并修改配置使应用连接本地数据库

- 虚拟机 `lain build` 用测试配置构建应用
- [TODO] 虚拟机 `lain web run` 会按照 `lain.yaml` 中 `web proc` 描述 serve 在指定端口，即可通过 http://192.168.10.24:8188 访问本地测试服务
    - console 本地运行需要仔细阅读 `apps/settings.py` 里面对于各环境变量的获取及其默认值，运行时应注入这些环境变量（暂时 lain cli 还未支持 coming soon）
- 虚拟机 mkvirtualenv console ; pip install -r pip-req.txt ; python manage.py runserver 0.0.0.0:18000，即可通过 http://192.168.10.24:18000 访问本地测试服务
    - console 本地运行需要仔细阅读 `apps/settings.py` 里面对于各环境变量的获取及其默认值，运行时应预置这些环境变量

### 2. upstream master continuous integration

>目前 job 是手工创建， Jenkins On Lain 完成之后会无缝切换为基于用户注册的自动 job 创建

- [job 群](http://jenkins.lain.bdp.cc/view/lain/)（自动job管理在 Lain 0.5.x 的 JenkinsOnLain 规划中）
    - [master mr](http://jenkins.lain.bdp.cc/view/lain/job/console--console__mr__test/)  主动检测 `console/console` 的 merge request list 及其代码变更，对 src 和 target 代码触发合并后的 `lain test`
    - [master release](http://jenkins.lain.bdp.cc/view/lain/job/console--console__master__release/)
        - 主动检测 `console/console:master` 代码变更，触发 `lain test`
        - 在上述 test 运行并返回测试成功的结果之后构建二进制产品并发布到 lain 的 dev registry
- [DON'T DO IT]虚拟机 `lain deploy dev`  在开发集群上测试部署此 App （console 是 lain 集群的基础组件，不建议用普通应用的部署方式来更新）
- 虚拟机 `lain deploy local`  作为 lain 的开发者可以部署到本地的 lain 集群进行测试

### 3. prod deployment

>前置条件是上面的 master test => master release 构建链的测试和镜像构建成功，具体版本 `SHA1` 会由 master release job 在 lain.yaml 标注的 slack 频道中发布

- 作为 lain 的基础组件 console 的发布是跟着 `lain/lain` 来发布
    - 将上述 `master__release` 形成的 meta 以及 release 镜像 `git pull` 到本地，更换镜像的前缀为 `docker.bdp.cc:5002` 并推送到 `docker.bdp.cc:5002` 这个私有 registry ，DEMO:
        - `docker pull registry.lain.bdp.cc/console:meta-1441771889-b528f0183238fae7f9dd7dccab7e491151862245`
        - `docker tag registry.lain.bdp.cc/console:meta-1441771889-b528f0183238fae7f9dd7dccab7e491151862245 docker.bdp.cc:5002/console:meta-1441771889-b528f0183238fae7f9dd7dccab7e491151862245`
        - `docker push docker.bdp.cc:5002/console:meta-1441771889-b528f0183238fae7f9dd7dccab7e491151862245`
        - `docker pull registry.lain.bdp.cc/console:release-1441771889-b528f0183238fae7f9dd7dccab7e491151862245`
        - `docker tag registry.lain.bdp.cc/console:release-1441771889-b528f0183238fae7f9dd7dccab7e491151862245 docker.bdp.cc:5002/console:meta-1441771889-b528f0183238fae7f9dd7dccab7e491151862245`
        - `docker push docker.bdp.cc:5002/console:release-1441771889-b528f0183238fae7f9dd7dccab7e491151862245`
    - FORK `lain/lain` ，并修改 `playbooks/roles/bootstrap-images/vars/main.yaml` 中 `console` 以及 `console-meta` 对应的镜像 TAG，提交 merge request
