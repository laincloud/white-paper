# Networkd

如果主机可以自己绑定 IP (比如说 QingCloud，AWS), 默认采用的是 Virtual IP 模式暴露服务的。
如果不支持(比如说阿里云)，默认采用的是 Node IP 模式，使用主机 IP 暴露服务给外部，当然这种模式受限于主机的端口数量。

## Node IP Mode

参考: [Node IP](nip.md)

## Virtual IP Mode

参考: [Virtual IP](vip.md)

## 维护

### 启动
```
systemctl start networkd
```

### 停止
```
systemctl stop networkd
```

### 日志
```
journalctl -u networkd -f -l
```
