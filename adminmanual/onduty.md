# 集群维护日常值班指南

日常值班内容：

- 监控系统重点指标，发现问题立刻解决
    - hedwig 可选组件，搭建起来之后自己建立集群的 screen
- 定期观察监控系统，注意  / 和 /data 以及 docker 存储的磁盘占用，发现问题趋势立刻解决
    - 观察（在 master 控制节点）
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
    - 在 master 控制节点的 lain 代码目录  `clean-node <node_name>:<node_ip>`
