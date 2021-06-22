---
title: "jwt认证"
date: 2021-06-02T22:49:18+08:00
draft: false
---

jwt是一种常见跨域认证方式，之前验证用户登陆通常使用session机制。

session是client访问server之后，server生成一个唯一ID返回给用户，sever可以用这个ID做为key存储一些用户信息（可能是已经登陆，或者一些用户状态信息）。之后用户访问问服务器的时候带上这个ID，服务器就可以根据ID来查到当前用户的一些状态信息了。

jwt是client访问server之后，server把一些用户信息做下签名，签名的方式是对（用户信息+密钥）进行加密。签名和用户拼接在一起当作token返回给client，client之后访问的时候带上token，server把token拆分成用户信息和签名，server用自己本地保存的密钥和用户信息再次加密（也就是再次签名）和用户传上来的密钥进行对比，如果一致则信息未被修改。server则可以信任token中的用户信息。



#### 优点

- jwt信息存储在client端，节省server的空间，而且session在多台server的时候需要做集中处理。

#### 缺点

- jwt的用户信息是base64编码的，不能存储敏感信息（或者再进行加密）
- jwt一旦签发就不受控制，在有效期内可以一直使用（或者通过黑名单的方式进行处理）

