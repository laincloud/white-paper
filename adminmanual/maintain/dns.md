# DNS

DNS 分为 2 部分，一个是集群的 DNS server Tinydns app, 另外一个是每个节点 Dnsmasq。

Dnsmasq 会劫持少数域名，其他的域名则转发给 Upstream server (包含 Tinydns app)

## Dnsmasq

### 配置

集群内部支持动态修改/劫持域名

在 `/lain/config/dnsmasq_addresses` 创建一个名为 `DOMAIN` 建，比如 `hello.com`, 其值为 `192.168.77.254`

每个节点的 networkd 就会自动修改节点的 Dnsmasq 配置，使其生效。

### 维护

#### 启动
```
systemctl start dnsmasq
```

#### 停止
```
systemctl stop dnsmasq
```

## Tinydns

Tinydns 是 .lain 的权威 DNS server，没有被 Dnsmasq 劫持的 .lain 域名都会转发给其处理。

### 配置
集群内部支持动态修改/劫持域名

在 `/lain/config/tinydns_fqdn` 创建一个名为 `DOMAIN` 建，比如 `test.lain`, 其值为 `[+test.lain:192.168.10.254:300]`

每个配置的值必须是一个 array 且每个 item 符合 tinydns 的语法

参考: [The tinydns-data program](https://cr.yp.to/djbdns/tinydns-data.html)
