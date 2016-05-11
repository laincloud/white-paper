# Lainlet

若怀疑 Lainlet 的 API 返回的数据不正确，即与 etcd 中的数据不一致，可重启解决。

```sh
systemctl restart lainlet
```

若 Lainlet 启动异常，可检查etcd是否正常运行:`systemctl status etcd`。
