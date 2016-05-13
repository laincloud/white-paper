# 集群高可用

## 1. cluster master HA

>能执行集群管理职能的节点称作 master ，需要消除 master 单点

1. lain 代码 直接 `git clone https://github.com/laincloud/lain.git`
2. 管理节点的 ssh 私钥 `/root/.ssh/lain`
3. 集群的 SSL 证书备份 `/root/.certs`
4. lainctl 命令行工具 `pypi install git+https://github.com/laincloud/lainctl.git`

只要具有了上述要素，节点即可执行 lainctl 指令进行集群管理

## 2. webrouter HA

见 [webrouter](/adminmanual/maintain/webrouter.html) 的 scale 一节

## 3. tinydns HA

直接对 tinydns App 进行 lain scale 

## 4. registry HA

1. 采用本地存储方案的 registry 无法 HA
2. 使用 moosefs 或者 OSS 或者 S3 存储的 registry 即可 HA ，同样是直接对 registry App 进行 lain scale 即可
