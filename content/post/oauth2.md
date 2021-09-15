---
title: "OAuth2授权框架"
date: 2021-09-15T10:25:53+08:00
draft: false
---

## 1.前言

OAuth2 是一个授权框架，或称授权标准。简单解释：允许用户让第三方应用访问该用户在某一网站上存储的私密的资源（如照片，视频，联系人列表），而无需将用户名和密码提供给第三方应用。

1.1应用场景

现在用户都很懒，注册论坛的时候需要填写一大堆个人信息都嫌麻烦，所以现在很多论坛都对接了微信登录，只要点击微信登录并授权之后，论坛就可以使用微信中的用户信息（比如手机号，头像，昵称等）来注册论坛了。这种操作的背后就是OAuth2。

大致流程就是用户登录微信，并授权给论坛网站权限，使论坛可以获取用户微信的一些个人信息（手机号，头像，昵称等）。

## 2.OAuth2角色

OAuth 2 标准中定义了以下几种角色

- 资源所有者（Resource Owner）

- 资源服务器（Resource Server）

- 授权服务器（Authorization Server）

- 客户端（Client）

### 资源所有者（Resource Owner）

在 OAuth 2 标准中，资源所有者即代表授权客户端访问本身资源信息的用户（User），也就是应用场景中的“用户”。客户端访问资源服务器的权限仅限于用户授权的“范围”（aka. scope，例如读取或写入权限）。

### 资源/授权服务器（Resource/Authorization Server）

资源服务器托管了受保护的用户资源信息，而授权服务器验证用户身份然后为客户端派发资源访问令牌。

在上述应用场景中，微信既是授权服务器也是资源服务器，个人信息即为资源（Resource）。而在实际工程中，不同的服务器应用往往独立部署，协同保护用户账户信息资源。

### 客户端（Client）

在 OAuth2 中，客户端即代表意图访问受限资源的第三方应用。在访问实现之前，它必须先经过用户者授权，并且获得的授权凭证将进一步由授权服务器进行验证。

在上述应用场景中，论坛网站就是客户端。

<!--如果没有特别说明，下文中将不对"应用"，“第三方应用”，“客户端”做出区分。这些都指Client-->

## 3.OAuth 2 的授权流程

![img](https://raw.githubusercontent.com/jeffmingup/image/master/img/202109151626306.png)

1. Authorization Request
   客户端向用户请求对资源服务器的`authorization grant`。
2. Authorization Grant（Get）
   如果用户授权该次请求，客户端将收到一个`authorization grant`。
3. Authorization Grant（Post）
   客户端向授权服务器发送它自己的客户端身份标识和上一步中的`authorization grant`，请求访问令牌。
4. Access Token（Get）
   如果客户端身份被认证，并且`authorization grant`也被验证通过，授权服务器将为客户端派发`access token`。授权阶段至此全部结束。
5. Access Token（Post && Validate）
   客户端向资源服务器发送`access token`用于验证并请求资源信息。
6. Protected Resource（Get）
   如果`access token`验证通过，资源服务器将向客户端返回资源信息。

## 4.客户端应用注册

在应用 OAuth 2 之前，你必须在授权方服务中注册你的应用。如 [Google Identity Platform](https://developers.google.com/identity/) 或者 [Github OAuth Setting](https://developer.github.com/apps/building-oauth-apps/creating-an-oauth-app/)，诸如此类 OAuth 实现平台中一般都要求开发者提供如下所示的授权设置项。

- 应用名称
- 应用网站
- 重定向URI或回调URL

重定向URI是授权方服务在用户授权（或拒绝）应用程序之后重定向供用户访问的地址，因此也是用于处理授权码或访问令牌的应用程序的一部分。

###  3.1 Client ID 和 Client Secret

一旦你的应用注册成功，授权方服务将以`client id`和`client secret`的形式为应用发布`client credentials`（客户端凭证）。`client id`是公开透明的字符串，授权方服务使用该字符串来标识应用程序，并且还用于构建呈现给用户的授权 url 。当应用请求访问用户的帐户时，`client secret`用于验证应用身份，并且必须在客户端和服务之间保持私有性。

## 5.授权许可（Authorization Grant）

如上文的抽象授权流程图所示，前四个阶段包含了获取`authorization grant`和`access token`的动作。授权许可类型取决于应用请求授权的方式和授权方服务支持的 Grant Type。OAuth 2 定义了四种 Grant Type，每一种都有适用的应用场景。

- Authorization Code
  结合普通服务器端应用使用。
- Implicit
  结合移动应用或 Web App 使用。
- Resource Owner Password Credentials
  适用于受信任客户端应用，例如同个组织的内部或外部应用。
- Client Credentials
  适用于客户端调用主服务API型应用（比如百度API Store）

以下将分别介绍这四种许可类型的相关授权流程。

### 5.1 Authorization Code Flow

`Authorization Code` 是最常使用的一种授权许可类型，它适用于第三方应用类型为`server-side`型应用的场景。`Authorization Code`授权流程基于重定向跳转，客户端必须能够与`User-agent`（即用户的 Web 浏览器）交互并接收通过`User-agent`路由发送的实际`authorization code`值。

![img](https://raw.githubusercontent.com/jeffmingup/image/master/img/202109151631129.png)

####  User Authorization Request

首先，客户端构造了一个用于请求`authorization code`的URL并引导`User-agent`跳转访问。

```
https://authorization-server.com/auth
 ?response_type=code
 &client_id=29352915982374239857
 &redirect_uri=https%3A%2F%2Fexample-client.com%2Fcallback
 &scope=create+delete
 &state=xcoiv98y2kd22vusuye3kch
```

- response_type=code
  此参数和参数值用于提示授权服务器当前客户端正在进行`Authorization Code`授权流程。
- client_id
  客户端身份标识。
- redirect_uri
  标识授权服务器接收客户端请求后返回给`User-agent`的跳转访问地址。
- scope
  指定客户端请求的访问级别。
- state
  由客户端生成的随机字符串，步骤2中用户进行授权客户端的请求时也会携带此字符串用于比较，这是为了防止`CSRF`攻击。

#### 2. User Authorizes Application

当用户点击上文中的示例链接时，用户必须已经在授权服务中进行登录（否则将会跳转到登录界面，**不过 OAuth 2 并不关心认证过程**），然后授权服务会提示用户授权或拒绝应用程序访问其帐户。以下是授权应用程序的示例：

![img](https://raw.githubusercontent.com/jeffmingup/image/master/img/202109151634688.png)

#### 3. Authorization Code Grant

如果用户确认授权，授权服务器将重定向`User-agent`至之前客户端提供的指向客户端的`redirect_uri`地址，并附带`code`和`state`参数（由之前客户端提供），于是客户端便能直接读取到`authorization code`值。

```
https://example-client.com/redirect
 ?code=g0ZGZmNjVmOWIjNTk2NTk4ZTYyZGI3
 &state=xcoiv98y2kd22vusuye3kch
```

`state`值将与客户端在请求中最初设置的值相同。客户端将检查重定向中的状态值是否与最初设置的状态值相匹配。这可以防止CSRF和其他相关攻击。

`code`是授权服务器生成的`authorization code`值。`code`相对较短，通常持续1到10分钟，具体取决于授权服务器设置。

#### 4. Access Token Request

现在客户端已经拥有了服务器派发的`authorization code`，接下来便可以使用`authorization code`和其他参数向服务器请求`access token`（POST方式）。其他相关参数如下：

- grant_type=authorization_code - 这告诉服务器当前客户端正在使用`Authorization Code`授权流程。
- code - 应用程序包含它在重定向中给出的授权码。
- redirect_uri - 与请求`authorization code`时使用的`redirect_uri`相同。某些资源（API）不需要此参数。
- client_id - 客户端标识。
- client_secret - 应用程序的客户端密钥。这确保了获取`access token`的请求只能从客户端发出，而不能从可能截获`authorization code`的攻击者发出。

#### 5. Access Token Grant

服务器将会验证第4步中的请求参数，当验证通过后（校验`authorization code`是否过期，`client id`和`client secret`是否匹配等），服务器将向客户端返回`access token`。

```
{
  "access_token":"MTQ0NjJkZmQ5OTM2NDE1ZTZjNGZmZjI3",
  "token_type":"bearer",
  "expires_in":3600,
  "refresh_token":"IwOGYzYTlmM2YxOTQ5MGE3YmNmMDFkNTVk",
  "scope":"create delete"
}
```

至此，授权流程全部结束。直到`access token` 过期或失效之前，客户端可以通过资源服务器API访问用户的帐户，并具备`scope`中给定的操作权限。

### 5.2 Implicit Flow

`Implicit`授权流程和`Authorization Code`基于重定向跳转的授权流程十分相似，但它适用于移动应用和 Web App，这些应用与普通服务器端应用相比有个特点，即`client secret`不能有效保存和信任。

相比`Authorization Code`授权流程，`Implicit`去除了请求和获得`authorization code`的过程，而用户点击授权后，授权服务器也会直接把`access token`放在`redirect_uri`中发送给`User-agent`（浏览器）。 同时第1步构造请求用户授权 url 中的`response_type`参数值也由 *code* 更改为 *token* 或 *id_token* 。

![img](https://raw.githubusercontent.com/jeffmingup/image/master/img/202109151634807.png)

#### 1. User Authorization Request

客户端构造的URL如下所示：

```
https://{yourOktaDomain}.com/oauth2/default/v1/authorize?client_id=0oabv6kx4qq6
h1U5l0h7&response_type=token&redirect_uri=http%3A%2F%2Flocalhost%3
A8080&state=state-296bc9a0-a2a2-4a57-be1a-d0e2fd9bb601&nonce=foo'
```

*response_type*的`response_type`参数值为 *token* 或 *id_token* 。其他请求参数与`Authorization Code`授权流程相比没有并什么变化。

#### 2. User Authorizes Application（略）

#### 3. Redirect URI With Access Token In Fragment

假设用户授予访问权限，授权服务器将`User-agent`（浏览器） 重定向回客户端使用之前提供的`redirect_uri`。并在 uri 的 #fragment 部分添加`access_token`键值对。如下所示：

```
http://localhost:8080/#access_token=eyJhb[...]erw&token_type=Bearer&expires_in=3600&scope=openid&state=state-296bc9a0-a2a2-4a57-be1a-d0e2fd9bb601
```

- token_type - 当且仅当`response_type`设置为 *token* 时返回，值恒为 *Bearer*。

> 注意在`Implicit`流程中，`access_token`值放在了 URI 的 #fragment 部分，而不是作为 ?query 参数。

#### 4. User-agent Follows the Redirect URI

`User-agent`（浏览器）遵循重定向指令，请求`redirect_uri`标识的客户端地址，**并在本地保留 uri 的 #fragment 部分的`access_token`信息**。

#### 5. Application Sends Access Token Extraction Script

客户端生成一个包含 token 解构脚本的 html页面，这个页面被发送给`User-agent`（浏览器），执行脚本解构完整的`redirect_uri`并提取其中的`access_token`（`access token`信息在第4步中已经被`User-agent`保存）。

#### 6. Access Token Passed to Application

`User-agent`（浏览器）向客户端发送解构提取的`access token`。

至此，授权流程全部结束。直到`access token` 过期或失效之前，客户端可以通过资源服务器API访问用户的帐户，并具备`scope`中给定的操作权限。

### 5.3 Resource Owner Password Credentials Flow

`Resource Owner Password Credentials`授权流程适用于用户与客户端具有信任关系的情况，例如设备操作系统或同一组织的内部及外部应用。用户与应用交互表现形式往往体现为客户端能够直接获取用户凭据（用户名和密码，通常使用交互表单）。

![img](https://raw.githubusercontent.com/jeffmingup/image/master/img/202109151634800.png)

#### 1. Resource Owner Password Credentials From User Input

用户向客户端提供用户名与密码作为授权凭据。

#### 2. Resource Owner Password Credentials From Client To Server

客户端向授权服务器发送用户输入的授权凭据以请求 `access token`。客户端必须已经在服务器端进行注册。

```
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=password&username=johndoe&password=A3ddj3w
```

- grant_type - 必选项，值恒为 *password*

#### 3. Access Token Passed to Application

授权服务器对客户端进行认证并检验用户凭据的合法性，如果检验通过，将向客户端派发 `access token`>

```
{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "token_type":"example",
  "expires_in":3600,
  "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
  "example_parameter":"example_value"
}
```

### 5.4 Client Credentials Flow

`Client Credential`是最简单的一种授权流程。客户端可以直接使用它的`client credentials`或其他有效认证信息向授权服务器发起获取`access token`的请求。

![img](https://raw.githubusercontent.com/jeffmingup/image/master/img/202109151634785.png)

两步中的请求体和返回体分别如下：

```
POST /token HTTP/1.1
Host: server.example.com
Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
```

- grant_type - 必选项，值恒为 *client_credentials*

```
{
       "access_token":"2YotnFZFEjr1zCsicMWpAA",
       "token_type":"example",
       "expires_in":3600,
       "example_parameter":"example_value"
}
```

## 参考资料及文献

1. [OAuth2深入介绍](https://www.cnblogs.com/Wddpct/p/8976480.html)
2. [RFC 6749-OAuth 2.0授权框架简体中文翻译](https://github.com/jeansfish/RFC6749.zh-cn)

## 名词中英文对照

| 英文                | 中文     |
| ------------------- | -------- |
| Authorization Grant | 授权许可 |
| Authorization Code  | 授权码   |
| Access Token        | 访问令牌 |
| Authorization       | 授权     |
| Authentication      | 认证     |