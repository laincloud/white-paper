# Lain-jenkins

lain-jenkins 主要是为了更好的自动化构建lain的组件，从而更快速有效同步lain中组件

# 1. Introduction

[lain-jenkins-config](https://github.com/laincloud/lain-jenkins-config) is jenkins' job build config, which would build, tag and push images to your own registry

lain-jenkins-config 用于[lain](https://github.com/laincloud/lain.git) 基础组件镜像的自动化构建，主要包括两种类型的组件 lain build 和 non-build组件，最终 更新lain

lain build组件都是通过 lain build && lain tag dev 和 lain push dev 三步构建镜像，并将对象push到指定registry

non-lain build 组件则是有自己的构建规则，然后最终push到指定registry


# 2. Jenkins build step

2.1 install docker with version 1.11.1 or later, start docker

2.2 install [lain-cli](https://github.com/laincloud/lain-cli) && [lain-sdk](https://github.com/laincloud/lain-sdk)

2.3 lain config (private_docker_registry & domain(eg: dev)) to your registry

2.4 install jenkins && run jenkins with **root**

2.5 docker login {registry}

2.6 config jenkins Root Url && config registry for jenkins

2.7 install plugin of jenkins

2.8 restart jenkins

2.9 create jenkins job



# 3 Run jenkins with root user

3.1 edit jenkins config file

```
vi /etc/sysconfig/jenkins

$JENKINS_USER="root"
change others as you like, such as port, address etc.

```

3.2 change ownership of jenkins

```
chown -R root:root /var/lib/jenkins
chown -R root:root /var/cache/jenkins
chown -R root:root /var/log/jenkins
```

3.3 restart jenkins

```
service jenkins restart
```

# 4 Tools in repo

4.1 centos_build.sh: build base environment of jenkins (step 2.1 to 2.4) (set env(PRIVATE_DOCKER_REGISTRY && REGISTRY) before run this shell)

4.2 jenkins_url.sh: set Root Url for Jenkins (step 2.6)

4.3 jenkins_registry.sh: set registry address in jenkins (step 2.6)

4.4 jenkins_util: install jenkins plugin and create jobs (rewrite {job}/config.xml as your like) (step 2.7 to 2.9)

# 5 Build jenkins with tools

5.1 set env (PRIVATE_DOCKER_REGISTRY && REGISTRY)

5.2 ./centos_build.sh

5.3 docker login ${registry} **#login for image push**

5.4 ./jenkins_url.sh **#set Root Url for Jenkins**

5.5 ./jenkins_registry.sh ${registry} **#config registry in jenkins**

5.6 change plugin.cfg and check config.xml as you like

5.7 ./jenkins_util -a {address} -u {username} -p {passward} -i -c


# 6 Attention
```
1. set env of PRIVATE_DOCKER_REGISTRY & REGISTRY
2. docker login
3. set jenkins root url
4. set registry address to 'non-lain build' component's config.xml 
```