# Hedwig

[Hedwig](https://github.com/laincloud/hedwig) 是一套为 Lain 集群服务的指标收集、汇总、展示、监控系统。

## Hedwig 在 lain 上的部署

1. 根据网络状况，决定是否将 hedwig 依赖的两个镜像 laincloud/centos-lain 和 laincloud/icinga2 拉到本地并推送至某个 lain 集群可达的 registry.
1. 修改 Hedwig 的 lain.yaml，可能的修改包括两个基础镜像的位置与版本号，以及所有 web proc 的 mountpoint. 
1. 部署 hedwig 至 lain 集群
1. 在 lain 集群上设置 hedwig 虚 ip
```
etcdctl set /lain/config/vips/*.*.*.* '{"app":"hedwig","proc":"carbon","ports":[{"dest":"2003","src":"2003","proto":"tcp"},{"dest":"8888","src":"8888","proto":"tcp"}]}'
etcdctl set /lain/config/vips/*.*.*.* '{"app":"hedwig","proc":"icinga","ports":[{"dest":"5665","src":"5665","proto":"tcp"}]}'
```
1. 重启 networkd, 例如
```
ansible -i playbooks/cluster all -m shell -a 'systemctl restart networkd'
```
1. 打开 http://hedwig-grafana.lain.local 配置报警项, 默认的管理员用户和密码为`admin admin`, 配置结束后最好保存配置。
