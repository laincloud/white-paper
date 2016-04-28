# sso 简介

sso 是 lain 的单点登录系统，为 lain 的应用管理用户, 提供用户注册、身份认证等系列功能.

sso 的用户分为两个层次，需要利用 sso 登录的用户，和需要 sso 管理自己的应用的用户.
后者需要了解更多关于 oauth2 和 openid connect 的知识.

## 用户注册
提供用户名、邮箱、自定义的密码即可.
注册后的用户，在 sso 看来是公开的，只要知道某人的用户名，
可以调用相关 api 查询该用户相关信息。
所以可以方便地查询某个人在 sso 的哪个组中。
如果用户的用户名或邮箱和已有用户冲突，则会提示用户已存在，
此时正确的流程是利用邮箱重置密码。

## 应用注册
注册后的用户，可以注册自己的应用。一般来说，
lain 上的应用是由 console 去自动帮用户注册的，所以下面的流程
是提供注册 oauth2 client 的方法.

- 用浏览器打开 https://`{LAIN_DOMAIN}`
- 点击`我的应用管理`，添加客户端.

正确添加客户端后，可以在 `我的应用列表` 中看到刚添加的应用.
sso 自动生成了 Client ID 和 Client Secret, 今后可以用 oauth2 标准协议，
让用户利用 sso 登录自己的应用.

## scope 
用户登录第三方应用时，会跳到 sso，这时候会有一个授权的列表，其实是该页面 URL 
中 scope 参数的值，表示授权被保护的资源. 当前已有的 scope 的列表如下：

- write:app
- read:app
- write:user
- read:user
- write:group
- read:group
- openid

openid 的含义见 openid connect 协议，如果 scope 包含 openid, 则用授权码得到的结果除了 access_token 外还有 id_token.

现在 sso 主要支持 oauth2 及 openid connect 的 code flow 和 implicit flow.

## Authentication code flow

- 首先注册应用
	- 可以在 sso 得网站上注册，也可以调用 api
	- POST /apps
- 客户端
	- 首先使用申请到的 "client id", "redirect uri" 以及其他参数跳转到 SSO 登陆页面， url 构造的 python 代码如下所示：
        ```
        url = 'https://sso.yxapp.xyz/oauth2/auth' + '?' + urlencode({'client_id': client_id, 'response_type': 'code', 'scope': 'write:group', 'redirect_uri': redirect_uri, 'state': '*****',})
        ```
    - 用户在SSO登陆界面登陆后会跳转到 redirect uri 并在链接末尾附带一个 code 参数
    - 客户端需截取 code 参数后附带 "client id", "client secret", "redirect uri" 等其他参数向 sso 请求 access_token，相关 url 构造的 python 代码如下所示：
        ```
        url = 'https://sso.yxapp.xyz/oauth2/token' + '?' + urlencode({'client_id': client_id, 'grant_type': 'authorization_code', 'client_secret': client_secret, 'code': code, 'redirect_uri': redirect_uri})
        ```
    - SSO 验证成功后会返回一个 http Response，status 为 201 则表明登录成功，其中 JSON 串中会包含 access_token


### 注意事项

```
如果应用包含前端与服务器端，则 client_secret 最好包含于服务器端，同时，服务器端在接到附带 access-token 的 http 请求时，应该先向 SSO 验证 access_token 的正确性（通过调用 https://sso.yxapp.syz/api/me/，在header中加上  "{'Authorization' : 'Bearer ' + access_token"}），再执行相应权限操作。
```

## Implicit flow
这里仅给出一种基于浏览器的前端实现的 demo.

类似常见的 login with google，lain 上的 sso 也可以作为 openid
connect 的身份认证服务器。

## 利用 implicit flow 认证

对于浏览器利用 sso 做认证的用例，最方便的方法是利用 openid connect 的 implicit flow, 使用步骤如下。

前提条件：需要在 sso 注册用户，并且有一个应用需要 sso 做认证。

* 打开 https://sso.`{LAIN_DOMAIN`}
* 在卡片“自助服务”下，点击“我的应用管理”，注册该应用及其回调 URL
* 用 implicit flow 进行认证

下面，假设 *LAIN_DOMAIN* 为 **lain.bdp.cc**, 结合一个例子详细说明第三步的使用方法。只需要简单的 4 步即可完成。

* 首先，需要引用 openid connect 的客户端 js 库
 
```html
<script src="https://sso.lain.bdp.cc/assets/oidc/oidc.js"></script>
```  

* 其次，配置 client，在 loginwithsso 按钮的响应函数中添加类似如下的代码，设置 client info，主要包括 client\_id, redirect\_uri; 另外，设置 sso 的配置，并保存在浏览器的 session 中： 

```js
    var clientInfo = {
        client_id: '3',  // sso 生成的 client_id  
        redirect_uri: 'http://' // 用户自己的回调 URL
    };
    OIDC.setClientInfo(clientInfo);

    var providerInfo = OIDC.discover("https://sso.lain.bdp.cc")
    OIDC.setProviderInfo(providerInfo);
    
    // 将配置存到浏览器 session 中 
    OIDC.storeInfo(providerInfo, clientInfo);
```

* 接下来，进行 login

```js
    OIDC.login({
        scope: 'openid', // 具体需要的 scope 与应用相关
        response_type: 'token id_token',
    });
```


* 最后，在用户回调的 URL 中，验证 id\_token 的签名及有效性，例如按照如下的 js 进行函数调用；

```js
    OIDC.restoreInfo();

    // Get ID Token: Returns ID Token if valid, else error. 
    var id_token = OIDC.getValidIdToken();
    alert("idtoken:" + id_token);

	// 得到解析后的 id_token
    var id_token_parts = OIDC.getIdTokenParts(id_token)
    alert(id_token_parts)

    // Get Access Token
    var token = OIDC.getAccessToken();
    alert("access_token:" + token)
```


