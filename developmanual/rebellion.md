# Rebellion
## Introduction
Rebellion 是 LAIN 中负责管理日志数据的 layer0 组件, 以 host 模式运行在 docker 上。但 Rebellion 并不是单进程的容器，而是由 supervisord 管理的多进程容器。Rebellion 主要包括两个部分：
- [lain-filebeat](https://github.com/laincloud/beats)：负责接受日志输入、处理并发送。
- Rebellion 程序：负责在启动时生成 lain-filebeat 的配置文件。并监听 lainlet，当集群配置更新或应用更新时，及时更新lain-filebeat 的配置文件并重启 lain-filebeat。

Rebellion 目前主要处理三类日志流：

- Docker 的 Syslog：Rebellion 通过只读挂载主机的 /var/log/messages，并通过 `parse_regex_fields` 解析，通过 `tag_lain_fields` 加工，然后输出至Kafka。

- LainApp 的 log文件：Rebellion 通过只读形式挂载主机的 /data/lain/volume 目录，读取应用配置的日志文件，然后输出至 Kafka。

- Webrouter 的 access log：Rebellion 通过读取 webrouter volume 出来的*.access.log文件，使用 `parse_regex_fields` 解析得到每个请求的结构化日志，然后发送给 Kafka。


## Build and Run

### Build image
已经直接配置了 docker hub 上的自动构建。

### Run
Rebellion 的运行是以 Host 模式运行的 Container，目前已经作为 Service 在 boostrap 时就已经部署并运行了。
Service 中运行的指令为：

```bash
/usr/bin/docker run \
          --name %n --rm --network=host \
          -e LAINLET_PORT={{ LAINLET_PORT }} \
          -v /data/lain/volumes/:/data/lain/volumes/:ro \
          -v /data/lain/rebellion/var/lib/filebeat/:/var/lib/filebeat \
          -v /data/lain/rebellion/logs/filebeat:/var/log/filebeat/ \
          -v /data/lain/rebellion/logs/supervisor:/var/log/supervisor/ \
          -v /var/log/:/var/log/syslog/:ro \
          laincloud/rebellion:{{ rebellion_version }}
```

运行时的参数说明如下
- `-e LAINLET_PORT={{ lainlet_port }}`

  设置环境变量NODE_NAME为主机的host_name，设置环境变量LAINLET_PORT, GRAPHITE_PORT为集群bootstrap时的lainlet_port和graphite_port。
- `-v /data/lain/volumes/:/data/lain/volumes/:ro`

  以只读模式挂载lain的volume目录。
- `-v /data/lain/rebellion/var/lib/filebeat/:/var/lib/filebeat`

  将 filebeat 的 registry 目录挂载到宿主机上，registry 记录了读取的文件的位置等重要信息，防止升级rebellion时丢失。
- `-v /data/lain/rebellion/logs/supervisor:/var/log/supervisor/` 

  挂载 Rebellion 本身的日志文件以及 filebeat 的标准输出文件。

- `-v /var/log/:/var/log/syslog/:ro` 

  以只读形式挂载 `/var/log/messages` 目录，读取 docker 的标准输出。

> 目前由于所有的日志都发送至Kafka，所以部署时集群应该在etcd中配置了/lain/config/kafka