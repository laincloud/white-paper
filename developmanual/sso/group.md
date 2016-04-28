
SSO 的第一版已经规定了了 group 的基本逻辑，包括 group 的管理员是谁，group 的成员逻辑。
并实现了操作 group 的一些基本 API.

#SSO V2 group 需求

1. 分层的权限管理
2. 组的复用

比如 lain 上有很多应用，有一些 layer1 应用比较核心，有一些 layer2 应用并不核心，但是需要有 lain 的管理员作为其 maintainer. 当前需认证的每个 layer1 应用都在 sso 中建了一个组，但是其用户是相同的，我们可以将公共的成员提取出一个组，比如叫 lainadmin，然后将其添加入所有 layer1 或 layer2 应用中.

建议用户管理组时慎重，使组的语义满足最初建组的语义，以免给依赖该组的组的管理员带来不必要的麻烦。

#sso group 设计

###需求：为了支持一些分层的权限管理，需要一个嵌套 group 的实现。

###前提 / 现状：
- sso 是一个 openid connect 的 server，自然也是 oauth2 的 server
- sso 中的 group 在 sso 里是某些用户的集合，可以为 sso 的应用 (client) 提供权限管理
	- group 代表一个资源，属于 group 的用户代表拥有这个资源；
	- group 中的用户分为两种，分为 admin 和 normal, 
	- 其中 admin 是组的管理者，在 sso，admin 拥有修改 group 及其成员的权利；
	- 应用也可以利用这两个角色来做权限管理，但是建议用以下方式做权限管理：
	- user 在某 group 中则认为有权限，不在则认为没有权限

## 目标
实现 group 的包含嵌套，所谓包含，指的是若不违反规则，可以将 Group A 加入 Group B，这时 B 为 A 的父节点，含义是所有 A 的成员均为 B 的成员.

组和组的包含关系只是手段，目标是组和成员的关系.

必须实现查询某用户所属的所有组的 API.

不需要实现一个组的所有用户的 api. (至少对于 backend group，禁止查询其成员列表）

最后，backend group 是原子的，不可向其内部添加任何成员组(其用户成员添加修改删除规则由接口定义)。

###可能的副作用
大组包含的小组的管理者修改小组成员，可能会影响大组。

## 模型
由于存在一个大组包含两个小组，也存在一个小组被包含在两个大组中，所以不能用有向树来对组关系进行建模，但是为了逻辑的清晰，我们不容许有回路，所以大的模型是一个 DAG (Directed Acyclic Graph). 而节点为 group 和 user. 将 group 看成是 group 和 user 的集合. B 为 A 的父节点则在 DAG  上表现为 B 到 A 的有向边.

在 DAG 上，为边增加一个属性，即 admin/normal, 将之前 group 与其 member 的关系延拓到 group 与 group 之间.

需要注意的一点是，加上上述属性后，该 DAG 就不能做一些优化，比如 group A 到 B 之前是不需要两条不同的路径的，而如果边有 admin/normal，则不能加去掉重复组的优化.

- 组的成员：该组可达的成员成员
- 组的 admin 成员：管理者即从该组只通过 admin 边可达的节点
- 组的 normal 成员：除 admin 外其他的组员 

组的 admin 可以修改该组，即包括删除组，增加子组，删除子组，增加成员，删除成员，但不可修改子组.

为了控制该 DAG 的复杂性，引入一个深度参数，含义为从某节点到叶节点(即用户节点)的最长的路径长度.

### 新增 api

添加子组

PUT /api/groups/{groupname}/group-members/{groupname}

删除子组

DELETE /api/groups/{groupname}/group-members/{groupname}


### 修改 api

查看子组

GET /api/groups/{groupname}
增加字段 group\_members


## 其他功能
为了防止嵌套层次太深，规定一个常数代表最大深度.
另外，该功能是否启用是可配置的.

