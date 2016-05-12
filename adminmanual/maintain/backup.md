# 备份服务

backup目前仅支持MooseFS这一种后端存储，所以，在使用backupd前，一定要确保集群中已引入[MooseFS](/adminmanual/maintain/moosefs.html)

## 初始化

```
ansible-playbook -i playbooks/cluster -e "role=backupd" playbooks/role.yaml
```

初始化backupd后，再add-node添加节点，新node都会自动启动backupd服务。


## backupd

启用backup服务后, 每个node节点上都会运行一个监听9002端口的`backupd.service`。

backupd也是使用systemd管理:

```sh
systemctl status backupd
```

## backup controller

controller是一个普通的lain app, 它负责分发备份任务给各个backupd。可通过`curl backupctl.{domain}/api/v2/system/cron/jobs`的方式看其是否正常工作。

## 没有预期备份的问题排查

1. 检查lain.yaml配置是否有误。
1. 访问`backupctl.{domain}/api/v2/system/cron/jobs`查看结果中是否有期望的备份任务。若没有，可尝试重启lainlet解决。
1. 检查应用所在的node上的backupd是否正常运行。
1. 检查应用所在node上的moosefs是否挂载成功，backupd默认使用`/mfs/lain/backup`作为备份目录。
1. 进入到backupctl的container内部, 检查container内部到外界node上的9001和9002端口是否能正常被访问。
