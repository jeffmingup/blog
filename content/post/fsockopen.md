---
title: "记录fsockopen引发的一些思考和小坑"
date: 2021-06-22T11:45:59+08:00
draft: false
---

​	在公司无意间看到一个异步http客户端的PHP代码，来了兴趣，进去一看是调用了**fsockopen**这个函数发送的http请求，代码差不多如下<fsockopen.php>:

```php
<?php
$fp = fsockopen("localhost", 80, $errno, $errstr, 30);
if (!$fp) {
    echo "$errstr ($errno)<br />\n";
} else {
    $out = "GET /index.php HTTP/1.1\r\n";
    $out .= "Host: localhost\r\n";
    $out .= "Connection: Close\r\n";
    $out .= "\r\n";
    fwrite($fp, $out);
    // while (!feof($fp)) {
    //     echo fgets($fp, 128);
    // }
    fclose($fp);
}
```

​	对，就是这样只是发送了http请求，并没有接受响应，很溜。猛的一看好像也没毛病，因为这个代码是发送通知一类的逻辑，所以对响应结果不敏感，报错也没事。

​	我本地也尝试了一下，环境是lnmp，服务端代码如下<index.php>：

```php
<?php
$a = 1;
while ($a <= 10) {
    file_put_contents("./index.log", date('Y-m-d H:i:s') . "\n", FILE_APPEND);
    sleep(1);
    $a++;
}
```

​	我尝试执行了一下php fsockopen.php

![image-20210622145559175](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622145559175.png)

​	我本地使用的是docker环境，执行之后发现，nginx日志记录499，php脚本并没有执行。

​	网上查了一下，499对应的是 “client has closed connection”。这是nginx定义的一个状态码，用于表示这样的错误：服务器返回http头之前，客户端就提前关闭了http连接。这也符合我们的预期，因为我们发完请求直接就关闭了。

​	解决方法：**proxy_ignore_client_abort  on;** 确定在客户端关闭连接时是否应关闭与代理服务器的连接，而不在等待响应。我这边在nginx的**server**块添加了这个配置。（为啥特意说是server块呢？我刚开始写在了http块，配置无效，这绝壁是个bug！！！坑了我好久，以后爬坑要多试一试，别光猜）

![image-20210622161444025](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622161444025.png)

​	再次运行，无效还是499。。。

![image-20210622162952655](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622162952655.png)

​	网上找了半天，发现proxy_ignore_client_abort只有作为代理服务器的时候才有用，php-fpm的这种fast-cgi协议是无效的😭，我这边又加了一层代理，用nginx反向代理我本地的lnmp服务。

![image-20210622163416640](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622163416640.png)

​	终于成功了！！！

![image-20210622163602318](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622163602318.png)

​	但是我之前网上爬坑的时候发现了一个新的问题。

![image-20210622163759745](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622163759745.png)

​	**ignore_user_abort** 默认值**false**，客户端断开之后脚本会终止。但是我却执行了10秒到结束。并没有断开啊。

​	想了一下，因为我是用nginx代理了lnmp服务，加上我设置了proxy_ignore_client_abort参数。所以lnmp的客户端是nginx，我脚本执行结束之后，nginx代理与lnmp服务器之间是没有断开的。为了证实这个想法，（因为本地抓不到docker环境内的数据包。我用golang重新写了个web服务，逻辑都是一样的）然后用nginx做反向代理。并抓了下包。

```go
package main

import (
	"fmt"
	"html"
	"log"
	"net/http"
	"time"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		for i := 0; i < 10; i++ {
			log.Println(i)
			time.Sleep(time.Second)
		}
		fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
	})

	log.Fatal(http.ListenAndServe(":82", nil))
}
```

​	

![image-20210622170917096](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622170917096.png)

​	本地端口60221  --->  nginx端口80

​	nginx反向代理请求go服务

​	Nginx端口60222 ---> go web端口82

​	(48-49)可以看到fsockopen.php发送完http请求就直接发送fin包表示断开了。

 	之后(52-5095) nginx作为代理服务器依然和go服务发起了三次握手，发送http请求，接受响应，四次挥手，走了完整流程。

​	(5090-5093)最后nginx作为代理服务器，还想通过80端口把go服务的响应返回给PHP脚本。脚本进程早就退出了，所以一直返回RST包（重置连接）。

​	这就说通了。**nginx 的proxy_ignore_client_abort开启之后，作为代理服务器，在客户端断开之后，依然会保持连接请求服务器资源。**所以这个参数很危险，已经没有客户端连接接受响应了却还在处理请求，浪费服务器资源，一般不要开，除非你知道你在干啥。

​	最后我直接通过curl访问lnmp服务。然后ctrl+c断开。ignore_user_abort还是没有起作用。。。

```php
<?php
ignore_user_abort(0);
file_put_contents("./index.log", ignore_user_abort() . "999999\n", FILE_APPEND);
$a = 1;
while ($a <= 10) {
    file_put_contents("./index.log", date('Y-m-d H:i:s') . "\n", FILE_APPEND);
    sleep(1);
    $a++;
}
```

![image-20210622174008707](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622174008707.png)

​	最后发现需要刷新php的缓冲区才行，代码如下

![image-20210622174427344](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622174427344.png)

```php
<?php
ignore_user_abort(0);
file_put_contents("./index.log", ignore_user_abort() . "999999\n", FILE_APPEND);
$a = 1;
while ($a <= 10) {
    file_put_contents("./index.log", date('Y-m-d H:i:s') . "\n", FILE_APPEND);
    echo "1";
    ob_flush();
    flush();
    sleep(1);
    $a++;
}
```

最后，脚本终于如期中断了！！！![image-20210622174311345](https://raw.githubusercontent.com/jeffmingup/image/master/img/image-20210622174311345.png)
