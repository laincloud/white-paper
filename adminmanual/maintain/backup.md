# 备份服务

backupd目前仅支持MooseFS这一种后端存储，所以，在使用backupd前，一定要确保[MooseFS](/sa/moosefs/)已被初始化

## 初始化

```
ansible-playbook -i /vagrant/playbooks/cluster -e "role=backupd" /vagrant/playbooks/role.yaml
```

初始化backupd后，再add-node添加节点，新node都会自动启动backupd服务
