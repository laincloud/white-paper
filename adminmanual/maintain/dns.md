# DNS

DNS 分为 2 部分，一个是集群的 DNS server Tinydns app, 另外一个是每个节点 Dnsmasq。

Dnsmasq 会劫持少数域名，其他的域名则转发给 Upstream server (包含 Tinydns app)

## Dnsmasq

### 配置
`/lain/config/dnsmasq_addresses`
### 维护


## Tinydns

### 配置
`/lain/config/tinydns_fqdn`
### 维护
