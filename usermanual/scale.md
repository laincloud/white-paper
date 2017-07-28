# Scale

用户可根据需要，使用 [cli](cli.html#scale) 对自己的 APP 进行资源配置升级或降级。

scale的内容包括 APP 的实例数量，每个实例的内存和CPU share。

## 实例数量

```sh
# 如，scale web proc 为4个实例
lain scale -n 4 PHASE web
```

将应用实例数量增加或减少到指定数量。

若是扩容，则所需时间较长，每个实例20~60秒。若是缩容，则耗时较短，每个实例大约3~10秒。

## 内存

```sh
# 如，scale web proc 为 512M 内存
lain scale -m 512M PHASE web
```
`lain.yaml` 中 proc 的 memory 配置了一个 proc 实例最大可用内存，默认 32M。 若在后期发现需要更大内存，可通过 scale 的方式增大内存，否则实例会因 `out of memory` 挂掉。

内存scale本质上是对 proc 的一次升级操作。需要对每个实例重启。中间可能会出现服务的不可用情况。

因为是升级，所以耗时也会久一些，而且与用户配置的setup_time值有关, 不考虑setup_time的情况下，每个实例需要大约30~100秒。

了解更多内容: [User memory constraints](https://docs.docker.com/engine/reference/run/#user-memory-constraints)

## CPU

 > 启用这个参数时需要确认 配置了节点资源 通过修改 /lain/config/resources 的配置
 ```sh
 etcdctl set /lain/config/resources {"cpu":8, "memory":"16G"}'
 ```
 
```sh
# 设置 CPU 最大占比上限
lain scale -c 1 PHASE web
```

这里设置的 CPU 值，并不是说让实例独占一个 CPU，或是最多可让实例占用 100% 的CPU，而是一个相对值。
每个应用能占用的CPU上限为节点CPU的一半，比如8核节点，容器最大CPU上限为400%。
CPU配置分为8个档，超过8的设置当被看做8，实际CPU上限为 ($配置档 / 8) * 节点CPU数 /2, 比如设置为4的8核节点的容器的CPU上限为 (4/8)*8/2 = 200%，
默认为0，对应2档。

详细内容请参考 [CPU Period Constraint](https://docs.docker.com/engine/reference/run/#cpu-period-constraint)
