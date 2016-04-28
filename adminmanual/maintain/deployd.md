# Deployd

## 启停deployd

```
systemctl start(stop) deployd
```

## 设定deployd的工作状态

Deployd正常工作时，会定时巡检集群的所有container，对错误状态的container就行自动纠正。当我们对集群需要维护调试时，可能有需要deployd进入维护态，以免对我们的调试工作造成影响。

进入维护态的deployd会停止巡检，而且通过Restful API发送的所有的变更类操作都会被暂存到内存队列中，等到恢复工作时才会被执行。

```
# 查看状态
curl http://deployd.lain:9003/api/status

# 设定为maintainence状态
curl -XPATCH http://deployd.lain:9003/api/status -H "Content-Type: application/json" -d '{"status":"stop"}'

# 设定正常工作状态
curl -XPATCH http://deployd.lain:9003/api/status -H "Content-Type: application/json" -d '{"status":"start"}'
```
