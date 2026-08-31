---
title: 搞懂 OAuth2 + OIDC + PKCE：从一条真实授权链接说起
date: 2026-03-21
---

最近在看一个 CLI 工具的登录流程，抓到了这样一条授权请求：

```
https://auth.openai.com/oauth/authorize?
  client_id=app_EMoamEEZ73f0CkXaXp7hrann
  &code_challenge=UsFd8nKJgN7J1HPXy5ednto7neuboHlJFH14bLsjC_o
  &code_challenge_method=S256
  &redirect_uri=http://localhost:1455/auth/callback
  &response_type=code
  &scope=openid+email+profile+offline_access
  &state=703a2e3719c8fde2dad10480346f7fb6
```

参数一堆，看起来花里胡哨。我本来只想快点拿到 token，结果越挖越发现自己之前对「授权码模式」的理解其实是半吊子。

这篇文章就把 OAuth2、OIDC、PKCE 这几个东西串起来讲清楚，尤其是 PKCE 到底在防什么。

### 名词解释

先把后文会反复出现的词对齐一下：

| 名词 | 一句话解释 |
| ------ | ----------- |
| **OAuth 2.0** | 授权协议，解决「应用能不能代替用户访问资源」 |
| **OIDC** | 建在 OAuth2 上的身份层，解决「这个用户是谁」 |
| **Authorization Code** | 授权码，用户同意后返回的一次性中间凭证，用来换 Token |
| **Access Token** | 访问令牌，拿去调受保护 API 用 |
| **ID Token** | 身份令牌（通常是 JWT），证明用户身份，OIDC 才有 |
| **Refresh Token** | 刷新令牌，Access Token 过期后用来换新的，一般要申请 `offline_access` |
| **PKCE** | Proof Key for Code Exchange，防止授权码被截获后冒用的机制 |
| **code_verifier** | 客户端本地生成的随机串，换 Token 时必须带上 |
| **code_challenge** | `code_verifier` 的哈希结果，授权请求时发给服务器 |
| **state** | 随机值，用来防 CSRF，确认回调是自己发起的 |
| **Discovery** | OIDC 自动发现机制，通过 `/.well-known/openid-configuration` 拿配置 |
| **JWKS** | JSON Web Key Set，一组公钥，用来验证 ID Token 签名 |

### OAuth2 和 OIDC 到底差在哪

很多人把它们混着说，其实分工很明确：

| 协议 | 主要解决什么 |
| ------ | ------------- |
| OAuth 2.0 | 授权：这个应用能不能代替用户访问某些资源 |
| OIDC（OpenID Connect） | 认证：这个用户到底是谁 |

OIDC 是建在 OAuth2 上面的。它多了一个 **ID Token**（通常是 JWT），里面带着用户身份信息。  
只拿 Access Token 只能调接口，并不代表你正式知道「登录的人是谁」。有了 ID Token，身份才算落地。

现在常见的「用某某账号登录」，底层基本都是 OIDC。

### 标准流程长什么样

以授权码模式 + PKCE 为例，大致是这样：

1. 客户端本地生成一个随机字符串 `code_verifier`
2. 算出 `code_challenge = BASE64URL(SHA256(code_verifier))`
3. 跳转到授权服务器，带上 `code_challenge`、`state`、`scope` 等参数
4. 用户登录并同意授权
5. 授权服务器重定向回客户端，带上 `code` 和原来的 `state`
6. 客户端用 `code` + 原始的 `code_verifier` 去换 Token
7. 服务器验证 `SHA256(code_verifier)` 是否等于之前存的 `code_challenge`，通过后返回 Access Token、ID Token，有时还有 Refresh Token

我抓到的那条链接，走的就是这个流程。回调回来时变成了：

```
http://localhost:1455/auth/callback?
  code=ac_mvIVkAYv8aJHc7Gvxi_FNkjgLHaX6qq1LffvTSIMqEc.OWTb9zFp3qv7hYyIdG3RHBi8HVEBhsDJER2_fbqBrmA
  &scope=openid+email+profile+offline_access
  &state=703a2e3719c8fde2dad10480346f7fb6
```

`state` 对得上，说明不是被伪造的回调。接下来客户端就要拿这个 `code` 去换真正的 Token 了。

### 为什么需要 PKCE？我一开始没想通的地方

PKCE 的核心验证就一句话：

```
hash(code_verifier) == code_challenge
```

我第一反应是：这哈希算法是公开的啊（一般是 SHA-256），那算出来肯定一致，这能防什么？

后来才反应过来——**安全不靠哈希保密，靠的是 `code_verifier` 本身不泄露**。

`code_challenge` 会随着授权请求发出去，服务器会存下来。  
但 `code_verifier` 一直只活在合法客户端的内存里，直到最后换 Token 的那一步才发送。

真正要防的攻击是「授权码被截获」。

在移动 App 或某些桌面场景里很常见：

- 登录成功后，授权服务器通过自定义 URL Scheme 把带 `code` 的链接重定向回来
- 恶意 App 也可以注册同一个 Scheme，把这个重定向抢走
- 于是它轻松拿到了 `code`

如果没有 PKCE，公共客户端又没有 `client_secret`，攻击者拿着偷来的 `code` 直接去 Token 接口换，基本就能成功。

有了 PKCE 之后，即使 `code` 被偷了，攻击者没有 `code_verifier`，换 Token 时验证直接失败。

所以：

- 偷 `code` 在某些环境下相对容易（重定向劫持）
- 直接偷最终 Token 难得多（它走的是后端 HTTPS 接口，不会出现在重定向 URL 里）

PKCE 的价值就是把「偷到 code 就能换 Token」变成「偷到 code 也没用」。

### Discovery 和 JWKS 顺便提一下

现代 OIDC 里还会碰到这两个：

- **Discovery**：访问 `/.well-known/openid-configuration`，能自动拿到授权端点、Token 端点、JWKS 地址等配置，不用写死。
- **JWKS**：一组公钥。ID Token 是用私钥签的，客户端下载 JWKS 里的公钥来验证签名，确认 Token 没被篡改、确实是这个发行方发的。

它们和 PKCE 不是同一层的东西，但经常一起出现。

### 常见疑问

**Q：前面换来换去那么多，不就是为了最后拿 Token 吗？为什么盯着 code 防？**

因为在真实攻击面里，code 暴露的机会比最终 Token 大。Token 交换发生在客户端和授权服务器之间的直接请求里，而 code 常常会经过一次浏览器/系统级的重定向，容易被截。

**Q：哈希是公开算法，那验证不是必过吗？**

对合法客户端必过。对攻击者不过，因为他根本拿不到原始的 `code_verifier`。SHA-256 是单向的，从 `code_challenge` 反推几乎不可行。

**Q：公共客户端为什么不直接用 client_secret？**

因为 secret 放在前端或 App 里等于公开。PKCE 就是给「无法安全保存 secret 的客户端」准备的替代方案。

**Q：state 和 PKCE 有什么区别？**

`state` 主要防 CSRF（确认回调是自己发起的那一次）。  
PKCE 防的是授权码本身被第三方截获后冒用。两个解决的问题不一样，经常一起用。

### 小结

这条看起来复杂的授权链接，其实就是在做一件事：  
在「用户已经同意」和「客户端真正拿到 Token」之间，加一道只有合法客户端才能跨过的门槛。

OAuth2 负责授权，OIDC 补上身份，PKCE 则把授权码模式在公共客户端上的漏洞补上。  
理解这三者之后，再看各种「登录」实现，就不会只觉得是一堆参数在跳来跳去了。
