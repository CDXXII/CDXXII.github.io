---
title: HTTP Authentication
date: 2018-11-03
---

本文简述了 HTTP 的几种认证方式。

## BASIC Authentication

BASIC 认证是从 HTTP/1.0 就定义的认证方式。现在已经没有太多网站使用这种认证方式。

 1. 客户端请求一个需要身份认证的页面,但是并没有提供用户名或者密码。
 ````http
 GET /private/ HTTP/1.1
 Host: www.google.com
````

2. 当客户端请求的资源需要 BASIC 认证的时候，服务器会随状态码 401 Authorization Required，返回带 WWW-Authenticate 首部字段的响应。该字段内包含认证的方式（BASIC）及 Request-URI 安全域字符串（realm）。
````http
HTTP/1.1 401 Authorization Required
Date: Mon, 19 Sep 2011 08:38:32 GMT
WWW-Authenticate: Basic realm="Input Your ID and Password."
````

3. 客户端接受到状态码 401 的客户端为了通过 BASIC 认证，需要将用户 ID 和密码构成，两者中间以 :（冒号）连接后，再经过 Base64 编码处理。假设用户的 ID 为 guest，密码也是 guest，连接起来就是 guest:guest。通过 Base64 编码，即 `btoa('guest:guest')`，最后的结果为 Z3Vlc3Q6Z3Vlc3Q=。把这段字符串介入首部字段 Authorization 后，发送请求。当用户代理为浏览器时，会弹出输入框，用户仅需要输入用户 ID 和密码即可。之后，浏览器会自动完成到 Base64 编码的转换工作。
 ````http
 GET /private/ HTTP/1.1
 Host: www.google.com
 Authorization: Basic Z3Vlc3Q6Z3Vlc3Q=
````

4. 服务器接受到包含首部字段 Authorization 的请求后，会对认证信息的正确性进行验证。如果验证通过，就会向客户端返回一条包含 Request-URI 资源的响应。
````http
HTTP/1.1 200 OK
Date: Mon, 19 Sep 2011 08:38:32 GMT
Server: Apache/2.2.3 (Unix)
````

Basic 认证采用 Base64 的编码方式。在 HTTP 等非加密通行的线路上进行 BASIC 认证的过程中，安全性极低。此外 BASIC 认证使用上不够灵活，我们认证成功后，一般的浏览器是无法实现注销操作的。




























