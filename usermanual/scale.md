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

> **建议不要使用这个设置，除非你很清楚这操作的影响。若服务出现性能问题，可通过增加实例的方式解决。**

> 因为目前 lain 还没有提供一个比较直观的法则来决定这个值设为多少才算合适。所以如果一定要设置这个值，需要与集群管理员商议，根据集群的实际资源使用情况来决定。

```sh
# 设置 CPU 占比
lain scale -c 1 PHASE web
```

这里设置的 CPU 值，并不是说让实例独占一个 CPU，或是最多可让实例占用 100% 的CPU。而是一个相对值，相对于同台机器上的其他实例，在 CPU 分配上的一个权重值，默认为0。

详细内容请参考 [CPU share constraint](https://docs.docker.com/engine/reference/run/#cpu-share-constraint)
