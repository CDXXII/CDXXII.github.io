---
title: HTTP Authentication
date: 2018-08-03
---

本文简述了 HTTP 的几种认证方式。

## BASIC Authentication

BASIC 认证是从 HTTP/1.0 就定义的认证方式。现在已经没有太多网站使用这种认证方式。

 1. 客户端请求一个需要身份认证的页面,但是并没有提供用户名或者密码。
 ````http
 GET /private/ HTTP/1.1
 Host: google.com
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
 Host: google.com
 Authorization: Basic Z3Vlc3Q6Z3Vlc3Q=
````

4. 服务器接受到包含首部字段 Authorization 的请求后，会对认证信息的正确性进行验证。如果验证通过，就会向客户端返回一条包含 Request-URI 资源的响应。
````http
HTTP/1.1 200 OK
Date: Mon, 19 Sep 2011 08:38:32 GMT
Server: Apache/2.2.3 (Unix)
````

Basic 认证采用 Base64 的编码方式。在 HTTP 等非加密通行的线路上进行 BASIC 认证的过程中，安全性极低。此外 BASIC 认证使用上不够灵活，我们认证成功后，一般的浏览器是无法实现注销操作的。

## DIGEST AUTHENTICATION

为了弥补 BASIC 认证存在的弱点，从 HTTP/1.1 起有了 DIGEST 认证方式，DIGEST 同样采用质询/响应的方式（challenge/response），但是不会像 BASIC 认证那样直接发送明文密码。所谓质询响应方式是指，一开始一方会先发送认证要求给另一方，接着使用从另外一方那里接受到的质询码计算生成响应码，最后将响应码返回给对方进行认证的方式。
 
 1. 同样，客户端请求一个需要认证的资源。
 ````http
 GET /digest/ HTTP/1.1
 Host: google.com
 ````
 
 2. 接受到客户端请求需要认证的资源时，服务器会随着状态码 401 Authorization Required，返回带 WWW-Authenticate 首部字段的响应。该字段内包含质问响应方式认证所需的临时质询码（随机数，nonce）。
 
 首部字段 WWW-Authenticate 内必须包含 realm 和 nonce 这两个字段的信息。客户端依靠向服务器回送这两个值进行认证。
 
 nonce 是一种每次随返回 401 响应生成的任意随机字符串。该字符串通常推荐由 Base64 编码的十六进制数的组成形式，但实际内容依赖服务器的具体实现。
 ````http
 HTTP/1.1 401 Authorization Required
 Host: google.com
 WWW-Authenticate: Digest realm="DIGEST", nonce="MOSQZ0itBAA=44abb6784cc9cbfc605a5b0893d36f23de95fcff", algorithm=MD5,qop="auth"
 ````
 
 3. 客户端收到 401 状态码的响应，响应中包含 DIGEST 认证必须的首部字段 Authorization 信息。首部字段 Authorization 内必须包含 username、realm、nonce、uri 和 response 的字段信息。
 其中，realm 和 nonce 就是之前从服务器接受到响应中的字段。username 是 realm 限定的范围内可认证的用户名。uri（digest-uri）即 Request-URI 的值。考虑到经代理转发都 Request-URI 的值可能被修改，因此事先会复制一份副本保存在 uri 内。response 也可叫做 Request-Digest，存放经过 MD5 运算后的密码字符串，形成响应码。
 ````http
  GET /digest/ HTTP/1.1
  Host: google.com
  Authorization: Digest username="guest", realm="DIGEST",nonce="MOSQZ0itBAA=44abb6784cc9cbfc605a5b0893d36f23de95fcff", uri="/digest/", algorithm=MD5, response="df56389ba3f7c52e9d7551115d67472f", qop=auth, nc=0000000001, cnonce="082c875dcb2ca740"
 ````
 
 4. 服务端接受到包含首部字段 Authorization 请求的服务器，会确认认证信息的正确性。认证通过后则返回包含 Request-URI 资源的响应。并且这时会在首部字段 Authentication-Info 写入一些认证成功的相关信息。DIGEST 认证提供了高于 BASIC 认证的安全等级，但是和 HTTPS 的客户端认证相比仍然很弱。DIGEST 认证提供反之密码被窃听的保护机制，但是不存在防止用户伪装的保护机制。
  ````http
  HTTP/1.1 200 OK
  Authentication-Info: rspauth="f218e9dd407a3d16f2f7d2c4097e900", cnonce="082c875dcbca740", nc=00000001, qop=auth
  ````

## Session Based Authentication

基于 Session 的认证方式是利用 session 和 cookies 来实现的认证方式。由于 HTTP 请求是无状态的，服务端需要在客户端发起认证请求的时候，创建 session 并通过 set-cookie 将生成的 session ID 发送给客户端。之后客户端每一次请求都会携带服务器写入的 cookie（session ID）发送给服务器。每次请求到达服务器的时候，会检查一下该客户端有没有在服务器创建 session 来判断是否认证成功。

1. 客户端发起认证请求，服务器会在客户端首次访问的时候在服务器创建 session，然后保存 session（通常会放在 redis 或者 mongo DB中）。接下来会通过这个 session 生成一个标识字字符串，并通过 Set-Cookie 的方式发送给客户端。

2. 浏览器发现响应头里面有 Set-Cookie 字段，就会将该值保存下来，下次发送请求的时候就会带上该 Cookie，

3. 服务器在接受客户端请求时会去解析请求首部的 cookie 中的 session ID，根据这个 session ID 来查找服务器端保存的该客户端的 session，然后判断该请求是否合法。

## Token Based Authentication

基于 Token 的认证方式是指，当客户端使用用户名和密码请求认证，服务端收到请求并验证成功后，会签发一个 Token，再把这个 Token 发送给客户端。客户端收到 Token 后可以把它储存在 Cookie 或者 Local Storage 里，之后的每次请求都需要携带服务端签发的 Token。服务端收到请求的时候，会去验证客户端请求里携带的 Token，如果验证成功，就会向客户端返回请求数据。

基于 Token 的认证方式，最常用的是 JWT（Json Web Token）。JWT 是通过对 Json 进行加密签名来实现的授权验证方案，通过登录成功后将相关信息组成 Json 对象，然后对这个对象进行指定方式的加密。

JWT 对象有三部分构成：
1. Header，其中包含两个字段，分别为typ（type）和 alg（algorithm）。其中 typ 是 token 的类型，这里固定为 JWT。alg 是使用的 hash 算法，例如：HMAC SHA256 或者 RSA。
````json
{
  "alg": "HS256",
  "typ": "JWT"
}
````

2. Payload，此部分存储我们需要传递的认证信息，包括用户名，密码，过期事件等原数据。JWT 规定了7个官方备选字段。分别是：iss (issuer)，签发人。exp (expiration time)，过期时间。sub (subject),主题。 aud (audience)，受众。nbf (Not Before)，生效时间。iat (Issued At)，签发时间。jti (JWT ID)，编号。
````json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
````

3. Signature ，这部分是对前两部分的签名，防止数据篡改。服务端需要指定一个密钥（secret），使用 Header 里面指定的签名算法（默认是 HMAC SHA256），按照下面的公式产生签名。
````
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
````

算出签名后，把 Header、Payload、Signature 三个部分拼成一个字符串，每个部分用（.）分割。另外 Header 和 Payload 加密时使用的是 Base64URL，这个算法和 Base64 基本类似。JWT 作为一个令牌（token），有些场合可能会放到 URL（比如 api.example.com/?token=xxx）。Base64 有三个字符+、/和=，在 URL 里面有特殊含义，所以要被替换掉：=被省略、+替换成-，/替换成_ 。这就是 Base64URL 算法。JWT 的最大缺点是，由于服务器不保存 session 状态，因此无法在使用过程中废止某个 token，或者更改 token 的权限。也就是说，一旦 JWT 签发了，在到期之前就会始终有效，除非服务器部署额外的逻辑。

使用 Token 和使用 Session 认证的区别在于，Session ID 只是一个唯一标示符，服务端需要通过这个字符串，来查询在服务端保持的 session 来确认用户的登录状态。但是 Token 本身就是一种登录成功的凭证。它是在登录成功后根据某种规则生成的信息凭证，内部保存这用户的登录状态。服务器只需要根据定义的规则校验即可。

## 参考资料

- 图解 HTTP
- [JSON Web Token 入门教程](http://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html)
