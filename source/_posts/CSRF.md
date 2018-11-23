---
title: CSRF
date: 2018-11-23 22:32:37
tags:
---

### 什么是CSRF

CSRF（Cross-site request forgery）跨站请求伪造：攻击者诱导受害者进入第三方网站，在第三方网站中，向被攻击网站发送跨站请求。利用受害者在被攻击网站已经获取的注册凭证，绕过后台的用户验证，达到冒充用户对被攻击的网站执行某项操作的目的。

##### 一个典型的CSRF攻击流程：

受害者登录a.com，并保留了登录凭证（Cookie）。  
攻击者引诱受害者访问了b.com。  
b.com 向 a.com 发送了一个请求：a.com/act=xx。浏览器会默认携带a.com的Cookie。  
a.com接收到请求后，对请求进行验证，并确认是受害者的凭证，误以为是受害者自己发送的请求。  
a.com以受害者的名义执行了act=xx。  
攻击完成，攻击者在受害者不知情的情况下，冒充受害者，让a.com执行了自己定义的操作。  

![avatar](./img/CSRF攻击.jpg)

### 分类

#### GET类型的CSRF

GET类型的CSRF利用非常简单，只需要一个HTTP请求，一般会这样利用：

```
<img src="http://bank.example/withdraw?amount=10000&for=hacker" >
```

在受害者访问含有这个img的页面后，浏览器会自动向http://bank.example/withdraw?account=xiaoming&amount=10000&for=hacker发出一次HTTP请求。bank.example就会收到包含受害者登录信息的一次跨域请求。

#### POST类型的CSRF

这种类型的CSRF利用起来通常使用的是一个自动提交的表单，如：

```
 <form action="http://bank.example/withdraw" method=POST>
    <input type="hidden" name="account" value="xiaoming" />
    <input type="hidden" name="amount" value="10000" />
    <input type="hidden" name="for" value="hacker" />
</form>
<script> document.forms[0].submit(); </script> 
```

访问该页面后，表单会自动提交，相当于模拟用户完成了一次POST操作。

>POST类型的攻击通常比GET要求更加严格一点，但仍并不复杂。任何个人网站、博客，被黑客上传页面的网站都有可能是发起攻击的来源，后端接口不能将安全寄托在仅允许POST上面。


#### 链接类型的CSRF

链接类型的CSRF并不常见，比起其他两种用户打开页面就中招的情况，这种需要用户点击链接才会触发。这种类型通常是在论坛中发布的图片中嵌入恶意链接，或者以广告的形式诱导用户中招，攻击者通常会以比较夸张的词语诱骗用户点击，例如：

```
<a href="http://test.com/csrf/withdraw.php?amount=1000&for=hacker" taget="_blank">
  重磅消息！！
<a/>
```

>由于之前用户登录了信任的网站A，并且保存登录状态，只要用户主动访问上面的这个PHP页面，则表示攻击成功。


#### 总结出CSRF的特点

- 攻击一般发起在第三方网站，而不是被攻击的网站。被攻击的网站无法防止攻击发生。
- 攻击利用受害者在被攻击网站的登录凭证，冒充受害者提交操作；而不是直接窃取数据。
- 整个过程攻击者并不能获取到受害者的登录凭证，仅仅是“冒用”。
- 跨站请求可以用各种方式：图片URL、超链接、CORS、Form提交等等。部分请求方式可以直接嵌入在第三方论坛、文章中，难以进行追踪。

>CSRF通常是跨域的，因为外域通常更容易被攻击者掌控。但是如果本域下有容易被利用的功能，比如可以发图和链接的论坛和评论区，攻击可以直接在本域下进行，而且这种攻击更加危险。


### 防护策略

基于CSRF的两个特点：

- CSRF（通常）发生在第三方域名。
- CSRF攻击者不能获取到Cookie等信息，只是使用。

针对这两点，我们可以专门制定防护策略，如下：

- 阻止不明外域的访问
 - 同源检测
 - Samesite Cookie

- 提交时要求附加本域才能获取的信息
 - CSRF Token
 - 双重Cookie验证

#### 同源检测

在HTTP协议中，每一个异步请求都会携带两个Header，用于标记来源域名：

- Origin Header
- Referer Header


但是，Origin在以下两种情况下并不存在：

- IE11同源策略： IE 11不会在跨站CORS请求上添加Origin标头，Referer头将仍然是唯一的标识。最根本原因是因为IE 11对同源的定义和其他浏览器有不同

- 302重定向： 在302重定向之后Origin不包含在重定向的请求中，因为Origin可能会被认为是其他来源的敏感信息。对于302重定向的情况来说都是定向到新的服务器上的URL，因此浏览器不想将Origin泄漏到新的服务器上。



***referer了解下：***  
对于Ajax请求，图片和script等资源请求，Referer为发起请求的页面地址。对于页面跳转，Referer为打开页面历史记录的前一个页面地址。因此我们使用Referer中链接的Origin部分可以得知请求的来源域名。

在以下情况下Referer没有或者不可信：

- IE6、7下使用window.location.href=url进行界面的跳转，会丢失Referer。  
- IE6、7下使用window.open，也会缺失Referer。  
- HTTPS页面跳转到HTTP页面，所有浏览器Referer都丢失。  
- 点击Flash上到达另外一个网站的时候，Referer的情况就比较杂乱，不太可信。  

***Host、Origin、Referer区别了解下：***

- Host：描述请求将被发送的目的地，包括，且仅仅包括域名和端口号。 
在任何类型请求中，request都会包含此header信息。

- Origin：用来说明请求从哪里发起的，包括，且仅仅包括协议和域名。 
这个参数一般只存在于CORS跨域请求中，可以看到response有对应的header：Access-Control-Allow-Origin。

- Referer：告知服务器请求的原始资源的URI，其用于所有类型的请求，并且包括：协议+域名+查询参数（注意，不包含锚点信息）。


#### SameSite Cookie

SameSite Cookie允许服务器要求某个cookie在跨站请求时不会被发送，从而可以阻止CSRF。但目前SameSite Cookie还处于实验阶段，并不是所有浏览器都支持。

#### CSRF Token

让我们想到了登录验证的问题。

思路：CSRF攻击之所以能够成功，是因为服务器误把攻击者发送的请求当成了用户自己的请求。那么我们可以要求所有的用户请求都携带一个CSRF攻击者无法获取到的Token。服务器通过校验请求是否携带正确的Token，来把正常的请求和攻击的请求区分开，也可以防范CSRF的攻击。

##### 原理

用户打开页面的时候，服务器需要给这个用户生成一个Token，该Token通过加密算法对数据进行加密，一般Token都包括随机字符串和时间戳的组合（显然在提交时Token不能再放在Cookie中了，否则又会被攻击者冒用）因此，为了安全起见Token最好还是存在服务器的Session中，之后在每次页面加载时，使用JS遍历整个DOM树，对于DOM中所有的a和form标签后加入Token。这样可以解决大部分的请求，但是对于在页面加载之后动态生成的HTML代码，这种方法就没有作用，还需要程序员在编码时手动添加Token。

- - -

##### 步骤
 + 将CSRF Token输出到页面中
 + 页面提交的请求携带这个Token
 + 服务器验证Token是否正确


***分布式校验了解下:***

在大型站点，使用Session存储CSRF Token会带来很大的压力。

访问单台服务器session是同一个。但是现在的大型网站中，我们的服务器通常不止一台，可能是几十台甚至几百台之多，甚至多个机房都可能在不同的省份，用户发起的HTTP请求通常要经过像Ngnix之类的负载均衡器之后，再路由到具体的服务器上，由于Session默认存储在单机服务器内存中，因此在分布式环境下同一个用户发送的多次HTTP请求可能会先后落到不同的服务器上，导致后面发起的HTTP请求无法拿到之前的HTTP请求存储在服务器中的Session数据，从而使得Session机制在分布式环境下失效，因此在分布式集群中CSRF Token需要存储在Redis之类的公共存储空间。


由于使用Session存储，读取和验证CSRF Token会引起比较大的复杂度和性能问题。

目前很多网站采用Encrypted Token Pattern方式。这种方法的Token是一个计算出来的结果，而非随机生成的字符串。这样在校验时无需再去读取存储的Token，只用再次计算一次即可。

>这种Token的值通常是使用UserID、时间戳和随机数，通过加密的方法生成。这样既可以保证分布式服务的Token一致，又能保证Token不容易被破解。


***双重Cookie验证了解下：***

在会话中存储CSRF Token比较繁琐，而且不能在通用的拦截上统一处理所有的接口。

那么另一种防御措施是使用双重提交Cookie。利用CSRF攻击不能获取到用户Cookie的特点，我们可以要求Ajax和表单请求携带一个Cookie中的值。

>每个请求最好都要修改cookies

#### 防止网页被Frame

我们通过在服务端设置HTTP头部中的X-Frame-Options信息，防止网页被Frame。
X-Frame-Options 响应头有三个可选的值：

```
DENY：页面不能被嵌入到任何iframe或frame中；
SAMEORIGIN：页面只能被本站页面嵌入到iframe或者frame中；
ALLOW-FROM：页面允许frame或frame加载。
```

在服务端设置的方式如下：

```
Java代码: response.addHeader("x-frame-options","SAMEORIGIN");
Nginx配置: add_header X-Frame-Options SAMEORIGIN
Apache配置: Header always append X-Frame-Options SAMEORIGIN
```

#### CSRF测试

通过CSRF测试。
>CSRFTester是一款CSRF漏洞的测试工具

#### CSP

Content-Security-Policy 内容安全策略。

CSP是一种由开发者定义的安全性政策性申明，通过CSP所约束的的规责指定可信的内容来源（可以指脚本、图片、iframe、fton、style等可能的远程资源）。通过CSP协定，让WEB处于一个安全的运行环境中。

为了使CSP可用, 你需要配置你的服务器返回 Content-Security-Policy HTTP头部

***示例:***


```
Content-Security-Policy: default-src 'self'; 只允许同源下的资源
script-src 'self';只允许同源下的js
script-src 'self' www.google-analytics.com ajax.googleapis.com;允许同源以及两个地址下的js加载
default-src 'none'; script-src 'self'; connect-src 'self'; img-src 'self'; style-src 'self';多个资源时,后面的会覆盖前面
```

当然你还可以再meta中使用：

```
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; img-src https://*; child-src 'none';">
```

还可以启用发送违规报告，你需要指定 report-uri 策略指令，并提供至少一个URI地址去递交报告：

```
Content-Security-Policy: default-src 'self'; report-uri http://reportcollector.example.com/collector.cgi

```

作为报告的JSON对象报告包含了以下数据：

- document-uri 发生违规的文档的URI。
- referrer 违规发生处的文档引用（地址）。
- blocked-uri  被CSP阻止的资源URI。如果被阻止的URI来自不同的源而非文档URI，那么被阻止的资源URI会被删减，仅保留协议，主机和端口号。
- violated-directive 违反的策略名称。
- original-policy 在 Content-Security-Policy HTTP 头部中指明的原始策略。

样本： 

```
Content-Security-Policy: default-src 'none'; style-src cdn.example.com; report-uri /_/csp-reports
```
