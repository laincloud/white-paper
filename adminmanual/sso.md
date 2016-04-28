# SSO 运维文档

sso 支持 HA，由于每个 sso 的 container 是无状态的，
唯一的 proc 为 web 类型，所以如果 container 挂掉，只需要重启即可。 

## sso 的注册

sso 的用户注册需要邮箱激活。问题是很多同学收不到邮件。
那么这时候可能会需要 SA 帮忙激活一下。步骤如下：

- 首先搞清楚注册哪个集群，找 DBA 去查 `user_register` 表, 查到正在注册用户的 `activation_code`.
- GET `https://{LAIN_DOMAIN}/api/activateuser?code={activation_code}`

## sso 的管理员

这里简要地介绍一下 sso 的管理员的相关问题。

sso 的管理员即 sso 中，在 admins 组里的成员。所以，sso 的管理员有两个可能的身份，
admin/normal, admin 身份即可以管理 admins 组.

sso 的管理员权限：

1. 可以查看所有的 sso 用户，删除用户（同时会从直接包含该用户的所有组中删除该用户），然而这些操作是否成功取决于用户接口的实现，比如当前的 sso-mysql 是可以的，而 sso-ldap 的用户管理不在 sso 这边，在 ldap 的管理者那里。 
1. 然而当前不支持删除组，即 sso 的管理员并没有其它组的管理权限，任何组的管理权限均只在该组的 admin 成员手中。

