# 集群维护日常值班指南

日常值班内容：

- 监控系统重点指标，发现问题立刻解决
    - TBD hedwig/hagrid 里默认 screen 和策略
- 定期观察监控系统，注意  / 和 /data 以及 docker 存储的磁盘占用，发现问题趋势立刻解决
    - 观察
        - 观察 重点分区使用情况 : ansible -i playbooks/cluster all -m shell -a "df -lha | grep 'rootfs\|\/data'"
        - 观察 docker 存储使用情况 : ansible -i playbooks/cluster all -m shell -a "docker info"
        - 观察 系统日志空间占用 : ansible -i playbooks/cluster all -m shell -a "ls -lha /var/log/messages\*"
        - 观察 calico 日志 : ansible -i playbooks/cluster all -m shell -a "ls -lhat /var/log/calico/felix\*| head -3"
    - 常规清理
	    - ansible -i playbooks/cluster all -m shell -a "echo > /var/log/calico/felix.log.1"
	    - ansible -i playbooks/cluster all -m shell -a "echo > /var/log/calico/felix.log"
	    - ansible -i playbooks/cluster all -m shell -a "rm /var/log/messages-*"
	- 如果实在紧张则做紧急处理，一般不用
	    - ansible -i playbooks/cluster all -m shell -a "echo > /var/log/messages"
- 定期清理历史数据
    - 在控制节点的 lain 代码目录  `clean-node <node_name>:<node_ip>`
- [已自动化] 每天定时的 jenkins slave 维护
    - login session refresh : lain login ${PHASE} 之后在 cron 中加入 `/usr/bin/lain refresh ${PHASE}`
    - slave clear : cron 中加入 `/root/slave_clear.sh`  # 注意替换 ${PHASE} 和 ${DOMAIN}
        ```
            #!/bin/bash
            set -xe
            curl http://jenkins.${DOMAIN}/quietDown
            for x in `docker ps -a |grep Dead|awk '{print $1}'`;do docker rm -v $x || true;done
            for x in `docker ps -a |grep Exited|awk '{print $1}'`;do docker rm -v $x || true;done
            for x in `docker images|grep ' meta'|awk '{print $1":"$2}'`;do docker rmi $x || true;done
            for x in `docker images|grep ' release'|awk '{print $1":"$2}'`;do docker rmi $x || true;done
            for x in `docker images|grep ' test'|awk '{print $1":"$2}'`;do  docker rmi $x || true;done
            for x in `docker images|grep ' build'|awk '{print $1":"$2}'`;do  docker rmi $x || true;done
            for x in `docker images|grep '<none>'|awk '{print $3}'`;do  docker rmi $x || true;done
            curl -X POST http://jenkins.${DOMAIN}/cancelQuietDown
        ```
- [已自动化] 每天早11点的集群状态快速 check : 见 lain ansible task `playbooks/roles/cron`

配套工具：

- hedwig / hagrid 组件 TBD
- lainctl 相关工具  TBD
- node clean 工具
- cluster dashboard TBD
