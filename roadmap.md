# RoadMap

## [3.0.0](https://github.com/laincloud/lain/milestones/LAIN%203.0.0)

- 安全性升级
  - docker user space

- 容器排布和资源分配
  - 资源预分配模型，以便于支持大数据计算
  - NODE LABEL 以及 proc LABEL 匹配部署
  - 资源控制支持 blkio 和 networkio
  - cpu 支持更实用的资源分配

- 日志
  - hekad 的后继跟进，看是否需要再选型
  - 日志的伪实时查询获取

- 持续交付
  - staging 机制
  - 在线 debug 机制的再设计
  - 本地环境对在线集群的完美模拟

- 应用特性
  - oneshot 类型 proc

## 2.X.X

- 2.0.0 freshmeat (current)
  - 完善文档
  - 升级 Docker 版本

- [2.1.0](https://github.com/laincloud/lain/milestones/LAIN%202.1.0) bacon
  - bugfix，修复开源后发现的一些 bug.
  - 一些紧要 feature 的增补,  [查看更多](https://github.com/laincloud/lain/milestones/LAIN%202.1.0)

## 1.0.0

- 已在大数据生产环境和开发环境正式部署
- 目前的使用状况比较稳定，只需要定时 cron 进行日志清理，以及每隔一段时间根据监控进行数据清理
