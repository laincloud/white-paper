# 域名和 https 访问配置

## LAIN 里几种不同的域名的说明

1. `*.lain` ： 集群内建域名，只有集群管理员和开发者才会需要使用的域名，应用开发者不需要了解，提供集群本身基础设施。通过内部 DNS 劫持实现解析
2. `LAIN 集群标准域名` ( lain cluster domain ) ，提供给内部应用开发者进行应用部署和管理，即提供给 `LAIN CLI` 使用。称之为 `集群标准域名` 。
    - bootstrap 时，LAIN 集群标准域名会使用默认值 `*.lain.local`
    - bootstrap 完成后，可以通过配置的方式（即下面的设置 `extra_domains` 的方式）使用自定制的 LAIN 集群标准域名
    - lain 的主域名（main domain）为 extra_domains 的列表中的第一个域名，这个域名会体现在所有 lain App 的容器的环境变量 `LAIN_DOMAIN` 里，同时也是 console 和 lainctl 为 APP 建立 sso 组的名称的前缀
3. 应用自己的外部服务域名。具体可见 [mountpoint](../usermanual/lainyaml.html#mountpoint) 部分文档
    - 默认来说集群会给其上部署的应用的 web Proc 绑定一个默认的域名，
    - 应用一般有自己的特别的域名比如 `gmail.com` 虽然是 google 家的产品，虽然也有 `mail.google.com` 但是很显然用 `gmail.com` 这个域名更有意义，所以更建议通过 mountpoint 机制给应用的 web 服务绑定一个特别的外部域名进行外网服务


## 切换 LAIN 集群标准域名，并提供其配套的 SSL 证书

>建议在 bootstrap 第一个节点时就进行切换，否则后续会增加不少麻烦
>假设想使用的自定义集群标准域名叫 `example.xyz`

- [bootstrap](install/) 第一个节点成功
- 准备好 `lain-box` 以便于对集群和应用进行管理操作
    - 已在 `lain-box` 上劫持了 `*.lain.local` 域名或者写了 `/etc/hosts` 将几个重要的集群组件域名指向了正确的 ip
    - 已在 `lain-box` 执行过  `lain config save local domain lain.local` 和 `lain config save-global private_docker_registry registry.lain.local`
- 搞定 `example.xyz` 域名，以及 `*.example.xyz` 的通配符 SSL 证书 
    - 找一个 DNS 服务商配置好 DNS 解析服务，将域名解析到正确的 ip
    - 将 SSL 证书部署到 bootstrap 节点的 webrouter volume 里
        - 在 bootstrap 节点上 `cp example.xyz.crt /data/lain/volumes/webrouter/webrouter.worker.worker/1/etc/nginx/ssl/`
        - 在 bootstrap 节点上 `cp example.xyz.key /data/lain/volumes/webrouter/webrouter.worker.worker/1/etc/nginx/ssl/`
- 配置 LAIN 集群的 `extra_domains` 配置
    - 这个 `/lain/config/extra_domains` 记录了 LAIN 集群除了 `*.lain` `*.lain.local` 之外还能以什么样的域名作为 集群标准域名
    - 在 bootstrap 节点上执行 `etcdctl set /lain/config/extra_domains '["example.xyz"]'`
- 配置 LAIN 集群的 `ssl` 配置，指定 https 服务使用的证书
    - 在 bootstrap 节点上执行 `etcdctl get /lain/config/ssl` 会看到返回 `{"*.lain":"web","*.lain.local":"web"}`
        - 这表示集群默认配置 bootstrap 之后使用自签名的 `web.crt` `web.key` 作为 `*.lain` 和 `*.lain.local` 这两个域名的 SSL 证书（当然肯定是不可能容过 SSL 校验的，这里只是作为内置的临时方案）
    - 在 bootstrap 节点上执行 `etcdctl set /lain/config/ssl '{"*.lain":"web","*.lain.local":"web","*.example.xyz":"example.xyz"}'`
        - 加入的配置 `"*.example.xyz":"example.xyz"` 的意思是告诉集群对 `*.example.xyz` 域名都用 `example.xyz.crt` 和 `example.xyz.key` 这一对证书
        - 这也可以加入精确匹配 `"blog.laincloud.com": "blog.laincloud.com"` 这样的配置，让 LAIN 集群单独对 `blog.laincloud.com` 一个域名使用 `blog.laincloud.com.crt` 和 `blog.laincloud.com.key` 这一对证书
    - 在 bootstrap 节点上执行 `docker-enter webrouter.worker.worker.v0-i1-d0` 进入 webrouter 容器，容器里执行 `supervisorctl restart watcher` 重新加载配置，然后 `exit` 退出容器
        - 让 webrouter 重新获取 ssl 配置，刷新 nginx 的配置文件
- 在 lain-box 中触发 `registry`、`console` 以及 `lvault` 的重新部署来让新的域名以及证书配置生效
    - git clone https://github.com/laincloud/console.git && cd console && lain deploy local && lain ps local   ## 需要等待几十秒生效
    - lain deploy local -t registry && lain ps local ## 需要等待几十秒生效
    - lain deploy local -t lvault && lain ps local ## 需要等待几十秒生效
    - 其他想让域名和对应 SSL 证书生效的应用也一样需要触发一次重新部署
- 在 lain-box 重新设定 LAIN 的集群标准域名
    - `lain config save example domain example.xyz`
    - `lain config save-global private_docker_registry registry.example.xyz`
- 接下来 `lain-box` 里面的 lain CLI 操作，就可以用 example 这个集群 alias 代替 DEMO 里的 local 这个集群 alias 了
    - 例如 `lain tag example` `lain push example` `lain deploy example` 等
- 打开 auth
    - 打开 auth 基本与 [auth](auth.md) 中描述无异，但是有一点需要注意，即 `lainctl auth open` 中的 `realm` 参数需要设置为 `http://console.example.xyz/api/v1/authorize/registry/`，`domain` 参数使用默认设置 `lain.local` 即可。

## 给 mountpoint 中指定的公网域名配置对应的 SSL 证书

>假设我们要给 `powerlain.com` 这个域名配置上已购买好的 `powerlain.key` 和 `powerlain.crt` 这套 SSL 证书

- 将 `powerlain.key` 和 `powerlain.crt` 这套 SSL 证书放到 webrouter 所在的 LAIN 节点的容器 volumes 里面去，让 webrouter 里面的 nginx 能拿到这些证书
    - volume 的位置以及 webrouter 的 scale 请参考 [这里](maintain/webrouter.html)
    - 样例： 
        - `cp powerlain.crt /data/lain/volumes/webrouter/webrouter.worker.worker/1/etc/nginx/ssl/`
            - 这里面的数字 1 代表是 webrouter 的 Proc 的 1 号容器实例，同理在 webrouter N 号容器实例的服务器上要把 1 换成 N
        - `cp powerlain.key /data/lain/volumes/webrouter/webrouter.worker.worker/1/etc/nginx/ssl/`
- 在 LAIN 节点上执行 `etcdctl get /lain/config/ssl` ，假设看到返回 `{"*.lain":"web","*.lain.local":"web"}`
- 在 LAIN 节点上执行 `etcdctl set /lain/config/ssl '{"*.lain":"web","*.lain.local":"web","powerlain.com":"powerlain"}'`
    - `"powerlain.com":"powerlain.com"` 这个对应关系的含义是 对 key ( `powerlain.com` ) 匹配的 nginx servername ，使用 value ( `powerlain` ) 对应的 SSL 证书 ( `powerlain.key` 和 `powerlain.crt` )
- 在 bootstrap 节点上执行 `docker-enter webrouter.worker.worker.v0-i1-d0` 进入 webrouter 容器，容器里执行 `supervisorctl restart watcher` 重新加载配置，然后 `exit` 退出容器
    - 让 webrouter 重新获取 ssl 配置，刷新 nginx 的配置文件
