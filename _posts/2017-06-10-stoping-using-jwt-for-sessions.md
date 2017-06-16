---
title:  别再使用JWT
category: Flaw
layout: post

---

![header](/images/stop-using-jwt-for-sessions/jwt.webp)    



## 要点

- 在Web应用中，使用JWT替代session并不是个好主意
- 适合JWT的使用场景




抱歉，当了回标题党。我并不认为JWT是辣鸡，只是它经常被误用。



### 什么是JWT



根据维基百科的[定义](https://en.wikipedia.org/wiki/JSON_Web_Token)，**JSON WEB Token** (**JWT**, 读作 [/dʒɒt/])，是一种基于JSON的、用于在网络上声明某种主张的令牌（token）。JWT通常由三部分组成: 头信息（header）, 消息体（payload）和签名（signature）。

头信息指定了该JWT使用的签名算法:

```
header = '{"alg":"HS256","typ":"JWT"}'
```

`HS256` 表示使用了 HMAC-SHA256 来生成签名。

消息体包含了JWT的意图：

```
payload = '{"loggedInAs":"admin","iat":1422779638}'//iat表示令牌生成的时间
```

未签名的令牌由`base64url`编码的头信息和消息体拼接而成（使用"."分隔），签名则通过私有的key计算而成：

```
key           = 'secretkey'  
unsignedToken = encodeBase64(header) + '.' + encodeBase64(payload)  
signature     = HMAC-SHA256(key, unsignedToken) 
```

最后在未签名的令牌尾部拼接上`base64url`编码的签名（同样使用"."分隔）就是JWT了：

```
token = encodeBase64(header) + '.' + encodeBase64(payload) + '.' + encodeBase64(signature) 

# token看起来像这样: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJsb2dnZWRJbkFzIjoiYWRtaW4iLCJpYXQiOjE0MjI3Nzk2Mzh9.gzSraSYS8EXBxLN_oWnFSRgCzcmJmMjLiuyu5CSpyHI 
```

JWT常常被用作保护服务端的资源（resource），客户端通常将JWT通过HTTP的`Authorization` header发送给服务端，服务端使用自己保存的key计算、验证签名以判断该JWT是否可信：

```
Authorization: Bearer eyJhbGci*...<snip>...*yu5CSpyHI
```



### 那怎么就误用了呢

近年来RESTful API开始风靡，使用HTTP header来传递认证令牌似乎变得理所应当，而单页应用（SPA）、前后端分离架构似乎正在促成越来越多的WEB应用放弃历史悠久的cookie-session认证机制，转而使用JWT来管理用户session。支持该方案的人认为：

1. **该方案更易于水平扩展**

确实如此，在cookie-session方案中，cookie内仅包含一个session标识符，而诸如用户信息、授权列表等都保存在服务端的session中。如果把session中的认证信息都保存在JWT中，在服务端就没有session存在的必要了。当服务端水平扩展的时候，就不用处理session复制（cookie Replication）/session黏连（Sticky cookie）或是引入外部session存储了。但实际上外部session存储方案已经非常成熟了（比如Redis），在一些Framework的帮助下（比如[spring-cookie](HTTP://docs.spring.io/spring-cookie/docs/current/reference/html5/guides/hazelcast-spring.html)和[hazelcast](https://hazelcast.com/)），session复制也并没有想象中的麻烦。所以除非你的应用访问量非常非常非常(此处省略N个非常）大，使用cookie-session配合外部session存储完全够用了。

2. **该方案可防护CSRF攻击**

跨站请求伪造[Cross-site request forgery](https://en.wikipedia.org/wiki/Cross-site_request_forgery)(简称CSRF, 读作 [sea-surf])是一种典型的利用cookie-session漏洞的攻击，这里借用[spring-security](https://docs.spring.io/spring-security/site/docs/current/reference/html/csrf.html)的一个例子来解释CSRF：

假设你经常使用bank.example.com进行网上转账，在你提交转账请求时bank.example.com的前端代码会提交一个HTTP请求:

```
POST /transfer HTTP/1.1
Host: bank.example.com
cookie: JsessionID=randomid; Domain=bank.example.com; Secure; HttpOnly
Content-Type: application/x-www-form-urlencoded

amount=100.00&routingNumber=1234&account=9876
```

你图方便没有登出bank.example.com，随后又访问了一个恶意网站，该网站的HTML页面包含了这样一个表单：

```
<form action="https://bank.example.com/transfer" method="post">
	<input type="hidden" name="amount" value="100.00"/>
	<input type="hidden" name="routingNumber" value="evilsRoutingNumber"/>
	<input type="hidden" name="account" value="evilsAccountNumber"/>
	<input type="submit" value="点击就送!"/>
</form>
```

你被“点击就送”吸引了，当你点了提交按钮时你已经向攻击者的账号转了100元。 现实中的攻击可能更隐蔽，恶意网站的页面可能使用Javascript自动完成提交。尽管恶意网站没有办法盗取你的session cookie（从而假冒你的身份），但恶意网站向bank.example.com发起请求时，你的cookie会被自动发送过去。

因此，有些人认为前端代码将JWT通过HTTP header发送给服务端（而不是通过cookie自动发送）可以有效防护CSRF。在这种方案中，服务端代码在完成认证后，会在HTTP response的header中返回JWT，前端代码将该JWT存放到Local Storage里待用，或是服务端直接在cookie中保存HttpOnly=false（否则前端Javascript代码无权从cookie中获取数据）的JWT。在向服务端发起请求时，用Javascript取出JWT，再通过header发送回服务端通过认证。由于恶意网站的代码无法获取bank.example.com的cookie/Local Storage中的JWT，这种方式确实能防护CSRF，但将JWT保存在cookie/Local Storage中可能会给另一种攻击可乘之机，我们一会详细讨论它：跨站脚本攻击——XSS。

3. **该方案更安全**

由于JWT要求有一个秘钥，还有一个算法，生成的令牌看上去不可读，不少人误认为该令牌是被加密的。但实际上秘钥和算法是用来生成签名的，令牌本身不可读仅是因为`base64url`编码，可以直接解码，所以如果JWT中如果保存了敏感的信息，相对cookie-session将数据放在服务端来说，更不安全。

除了以上这些误解外，使用JWT管理session还有如下缺点：

1. 更多的空间占用

如果将原存在服务端session中的各类信息都放在JWT中保存在客户端，可能造成JWT占用的空间变大，需要考虑cookie的空间限制等因素，如果放在Local Storage，则可能受到XSS攻击。

2. 更不安全

这里是特指将JWT保存在Local Storage中，然后使用Javascript取出后作为HTTP header发送给服务端的方案。在Local Storage中保存敏感信息并不安全，容易受到跨站脚本攻击，跨站脚本（Cross site script，简称xss）是一种“HTML注入”，由于攻击的脚本多数时候是跨域的，所以称之为“跨域脚本”，这些脚本代码可以盗取cookie或是Local Storage中的数据。可以从这篇文章查看[XSS攻击](HTTP://www.cnblogs.com/luminji/archive/2012/05/22/2507185.html)的原理解释。

3. 无法作废已颁布的令牌

所有的认证信息都在JWT中，由于在服务端没有状态，即使你知道了某个JWT被盗取了，你也没有办法将其作废。在JWT过期之前（你绝对应该设置过期时间），你无能为力。

4. 不易应对数据过期

与上一条类似，JWT有点类似缓存，由于无法作废已颁布的令牌，在其过期前，你只能忍受“过期”的数据。

看到这里后，你可能发现，将JWT保存在Local Storage中，并使用JWT来管理session并不是一个好主意，那有没有可能”正确"地使用JWT来管理session呢？比如：
a) 不再使用Local Storage存储JWT，使用cookie，并且设置HttpOnly=true，这意味着只能由服务端保存以及通过自动回传的cookie取得JWT，以便防御XSS攻击
b) 在JWT的内容中加入一个随机值作为CSRF令牌，由服务端将该CSRF令牌也保存在cookie中，但设置HttpOnly=false，这样前端Javascript代码就可以取得该CSRF令牌，并在请求API时作为HTTP header传回。服务端在认证时，从JWT中取出CSRF令牌与header中获得CSRF令牌比较，从而实现对CSRF攻击的防护
c) 考虑到cookie的空间限制（大约4k左右），在JWT中尽可能只放”够用”的认证信息，其他信息放在数据库，需要时再获取，同时也解决之前提到的数据过期问题

这个方案看上去是挺不错的，恭喜你，你**重新发明**了cookie-session，可能实现还不一定有现有的好。



### 那究竟JWT可以用来做什么

事实上，JWT更适合一次性操作的认证:

> 服务B你好, 服务A告诉我，我可以操作<JWT内容>, 这是我的凭证（即JWT）

现在回想起来，我以前参与过的一个项目就有类似的场景，我们有两个团队正在将一个遗留预订系统的支付功能拆分为一个单独的服务，支付服务的目标是提供一个API，可以为一笔订单发起一次付款。在验收规格中有一条是只有订单进入“可付款”状态时，才能允许用户付款，而支付服务并没有足够的数据来验证发起支付的订单是否符合前置条件。当时的解决方案是由遗留预订系统原封不动的包装了支付服务的API，因为预订系统有足够的上下文来判断是否可以发起支付。现在回想起来真是一个糟糕的决定，因为任何对支付服务API的修改，都需要对遗留订单系统上再修改一遍，将服务拆分的收益抵消了大半。如果当时知道有JWT这项技术，可以考虑由遗留预订系统（服务A）在满足订单支付条件时，颁发一个过期时间很短的JWT给前端Web应用，前端Web应用也不用保存该JWT，直接将JWT放到Http header中向支付服务（服务B）的API发起请求，支付服务验证该JWT的签名即可以完成认证（证明遗留预订系统已经认可了订单状态可支付）。

![header](/images/stop-using-jwt-for-sessions/otg.png)

> 遗留预订系统仍然使用传统的cookie-session来管理用户session，而新的支付服务则是无状态的，通过JWT来鉴定用户以及是否有权限支付

### 总结

1. 在Web应用中，别再把JWT当做session使用，绝大多数情况下，传统的cookie-session机制工作得更好
2. JWT适合一次性的命令认证，颁发一个有效期极短的JWT，即使暴露了危险也很小，由于每次操作都会生成新的JWT，因此也没必要保存JWT，真正实现无状态。




### 参考
https://auth0.com/blog/ten-things-you-should-know-about-tokens-and-cookies/
https://auth0.com/docs/security/store-tokens#where-to-store-your-jwts
http://cryto.net/~joepie91/blog/2016/06/13/stop-using-jwt-for-sessions/    


