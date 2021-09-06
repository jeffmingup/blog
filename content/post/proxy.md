---
title: "docker容器内部使用代理"
date: 2021-09-06T21:40:48+08:00
draft: false
---

**Docker容器内需要翻墙的话，一般有两种方法**

(1)  **在~/.docker/config.json 加上配置信息，代理服务器的话不能用127.0.0.1，因为容器内和宿主机一般是两个网络环境。可以使用宿主机当前局域网的ip地址。**



```json
{
 "proxies":
 {
   "default":
   {
     "httpProxy": "http://192.168.1.12:7890",
     "httpsProxy": "http://192.168.1.12:7890",
     "noProxy": "*.test.example.com,.example2.com,localhost,127.0.0.0/8"
   }
 }
}
```





(2)  **build 或者 run 的时候使用--env 加参数，再或者在dockerfile中加上也行**



ENV HTTP_PROXY http://192.168.1.7:7890

ENV HTTPS_PROXY http://192.168.1.7:7890



***附注: 如果是golang项目的话，也可以是用go mod 代理。dockerfile中加上***

ENV GO111MODULE on

ENV GOPROXY https://goproxy.io,direct
