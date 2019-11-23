---
title: ajax和XMLHttpRequest
date: 2017-7-16 14:44:03
tags: 技术
---

Ajax and XMLHttpRequest

ajax是一种技术方案，但并不是一种新技术。它依赖的是现有的CSS/HTML/Javascript，而其中最核心的依赖是浏览器提供的XMLHttpRequest对象，是这个对象使得浏览器可以发出HTTP请求与接收HTTP响应。

XMLHttpRequest一开始只是微软浏览器提供的一个接口，后来各大浏览器纷纷效仿也提供了这个接口，再后来W3C对它进行了标准化，提出了XMLHttpRequest标准。XMLHttpRequest标准又分为Level 1和Level 2。

XMLHttpRequest Level 1主要存在以下缺点：
  ● 受同源策略的限制，不能发送跨域请求；
  ● 不能发送二进制文件（如图片、视频、音频等），只能发送纯文本数据；
  ● 在发送和获取数据的过程中，无法实时获取进度信息，只能判断是否完成；

Level 2对Level 1进行了改进，XMLHttpRequest Level 2中新增了以下功能：
  ● 可以发送跨域请求，在服务端允许的情况下；
  ● 支持发送和接收二进制数据；
  ● 新增formData对象，支持发送表单数据；
  ● 发送和获取数据时，可以获取进度信息；
  ● 可以设置请求的超时时间；


###### 下面细说XMLHttpRequest这个东西。

先来一段使用XMLHttpRequest发送Ajax请求的简单示例。

```
function ajax() {
    //构造表单数据
    var formData = new FormData();
    formData.append('aaa', 'bbb');
    //创建xhr对象 
    var xhr = new XMLHttpRequest();
    //设置xhr请求的超时时间
    xhr.timeout = 3000;
    //设置响应返回的数据格式
    xhr.responseType = "";
    //创建一个 post 请求，采用异步
    xhr.open('POST', '/xxx', true);
    //注册相关事件回调处理函数
    xhr.onload = function(e) { 
        if(this.status == 200||this.status == 304){
            console.log(this.responseText);
        }
    };
    xhr.ontimeout = function(e) { };
    xhr.onerror = function(e) {  };
    xhr.upload.onprogress = function(e) {};
    //发送数据
    xhr.send(formData);
}
```

##### 说说你平时可能忽略的知识点：

*如何设置request header
在发送Ajax请求时，我们可能需要设置一些请求头部信息。
比如content-type、connection、cookie、accept-xxx等。xhr提供了setRequestHeader来允许我们修改请求 header。
void setRequestHeader(DOMString header, DOMString value);

注意：
  ● 方法的第一个参数 header 大小写不敏感，即可以写成content-type，也可以写成Content-Type，甚至写成content-Type;
  ● Content-Type的默认值与具体发送的数据类型有关
  ● setRequestHeader必须在open()方法之后，send()方法之前调用，否则会抛错；
  ● setRequestHeader可以调用多次，最终的值不会采用覆盖override的方式，而是采用追加append的方式。

示例：
```
var client = new XMLHttpRequest();
client.open('GET', 'xxx.action');
client.setRequestHeader('X-Test', 'bbb');
client.setRequestHeader('X-Test', 'aaa');
//最终request header中"X-Test"为: bbb, aaa
client.send();
```

*如何获取response header
xhr提供了2个用来获取响应头部的方法：getAllResponseHeaders和getResponseHeader。前者是获取 response 中的所有header 字段，后者只是获取某个指定 header 字段的值。
另外，getResponseHeader(header)的header参数不区分大小写。
DOMString getAllResponseHeaders();
DOMString getResponseHeader(DOMString header);


注意：
  1. 使用getResponseHeader()获取某个 header 的值时，浏览器抛错Refused to get unsafe header "XXX"
  ● 原因1：W3C的 xhr 标准中做了限制，规定客户端无法获取 response 中的 Set-Cookie、Set-Cookie2(启用)这2个字段，无论是同域还是跨域请求；
  ● 还有一些其他的限制内的字段
  ● 原因2：W3C 的 cors 标准对于跨域请求也做了限制，规定对于跨域请求，客户端允许获取的response header字段只限于“simple response header”和“Access-Control-Expose-Headers” 。

"simple response header"包括的 header字段有：Cache-Control,Content-Language,Content-Type,Expires,Last-Modified,Pragma;

"Access-Control-Expose-Headers"：首先得注意是"Access-Control-Expose-Headers"进行跨域请求时响应头部中的一个字段，对于同域请求，响应头部是没有这个字段的。这个字段中列举的 header 字段就是服务器允许暴露给客户端访问的字段。


所以getAllResponseHeaders()只能拿到限制以外（即被视为safe）的header字段，而不是全部字段；而调用getResponseHeader(header)方法时，header参数必须是限制以外的header字段，否则调用就会报Refused to get unsafe header的错误。


*如何指定xhr.response的数据类型
有些时候我们希望xhr.response返回的就是我们想要的数据类型。比如：响应返回的数据是纯JSON字符串，但我们期望最终通过xhr.response拿到的直接就是一个 js 对象，该怎么实现？

有2种方法可以实现，一个是level 1就提供的overrideMimeType()方法，另一个是level 2提供的xhr.responseType属性。

xhr.overrideMimeType()
overrideMimeType是xhr level 1就有的方法，所以浏览器兼容性良好。这个方法的作用就是用来重写response的content-type.

这样做有什么意义呢？比如：server 端给客户端返回了一份document或者是 xml文档，我们希望最终通过xhr.response拿到的就是一个DOM对象，那么就可以用xhr.overrideMimeType('text/xml; charset = utf-8')来实现。


xhr.responseType

responseType是xhr level 2新增的属性，用来指定xhr.response的数据类型，目前还存在些兼容性问题那么responseType可以设置为哪些格式呢，如下：
值 xhr.response 数据类型 说明  
"" String字符串 默认值(在不设置responseType时)  
"text" String字符串   
"document" Document对象 希望返回 XML 格式数据时使用  
"json" javascript 对象 存在兼容性问题，IE10/IE11不支持  
"blob" Blob对象   
"arrayBuffer" ArrayBuffer对象   


下面是同样是获取一张图片的代码示例，相比xhr.overrideMimeType,用xhr.response来实现简单得多。
```
var xhr = new XMLHttpRequest();
xhr.open('GET', '/path/to/image.png', true);
//可以将`xhr.responseType`设置为`"blob"`也可以设置为`" arrayBuffer"`
//xhr.responseType = 'arrayBuffer';
xhr.responseType = 'blob';

xhr.onload = function(e) {
  if (this.status == 200) {
    var blob = this.response;
    ...
  }
};
xhr.send();
```

#### 总结：
虽然在xhr level 2中，2者是共同存在的。xhr.responseType功能更强大，xhr.overrideMimeType()能做到的xhr.responseType都能做到。所以我们现在完全可以摒弃使用xhr.overrideMimeType()了。

*如何获取response数据
xhr提供了3个属性来获取请求返回的数据，分别是：xhr.response、xhr.responseText、xhr.responseXML
  ● xhr.response
      ○ 默认值：空字符串""
      ○ 当请求完成时，此属性才有正确的值
      ○ 请求未完成时，此属性的值可能是""或者 null，具体与 xhr.responseType有关：当responseType为""或"text"时，值为""；responseType为其他值时，值为 null
  ● xhr.responseText
      ○ 默认值为空字符串""
      ○ 只有当 responseType 为"text"、""时，xhr对象上才有此属性，此时才能调用xhr.responseText，否则抛错
      ○ 只有当请求成功时，才能拿到正确值。以下2种情况下值都为空字符串""：请求未完成、请求失败
  ● xhr.responseXML
      ○ 默认值为 null
      ○ 只有当 responseType 为"text"、""、"document"时，xhr对象上才有此属性，此时才能调用xhr.responseXML，否则抛错
      ○ 只有当请求成功且返回数据被正确解析时，才能拿到正确值。以下3种情况下值都为null：请求未完成、请求失败、请求成功但返回数据无法被正确解析时

*如何追踪ajax请求的当前状态
在发一个ajax请求后，如果想追踪请求当前处于哪种状态，该怎么做呢？
用xhr.readyState这个属性即可追踪到。这个属性是只读属性，总共有5种可能值，分别对应xhr不同的不同阶段。每次xhr.readyState的值发生变化时，都会触发xhr.onreadystatechange事件，我们可以在这个事件中进行相关状态判断。
```
  xhr.onreadystatechange = function () {
    switch(xhr.readyState){
      case 1://OPENED
        //do something
            break;
      case 2://HEADERS_RECEIVED
        //do something
        break;
      case 3://LOADING
        //do something
        break;
      case 4://DONE
        //do something
        break;
    }
```
```
0 UNSENT (初始状态，未打开) 此时xhr对象被成功构造，open()方法还未被调用
1 OPENED (已打开，未发送) open()方法已被成功调用，send()方法还未被调用。注意：只有xhr处于OPENED状态，才能调用xhr.setRequestHeader()和xhr.send(),否则会报错
2 HEADERS_RECEIVED(已获取响应头) send()方法已经被调用, 响应头和响应状态已经返回
3 LOADING (正在下载响应体) 响应体(response entity body)正在下载中，此状态下通过xhr.response可能已经有了响应数据
4 DONE (整个数据传输过程结束) 整个数据传输过程结束，不管本次请求是成功还是失败
```

*如何设置请求的超时时间
如果请求过了很久还没有成功，为了不会白白占用的网络资源，我们一般会主动终止请求。XMLHttpRequest提供了timeout属性来允许设置请求的超时时间。
xhr.timeout
单位：milliseconds 毫秒
默认值：0，即不设置超时
我们知道从请求开始 算起，若超过 timeout 时间请求还没有结束（包括成功/失败），则会触发ontimeout事件，主动结束该请求。

那么到底什么时候才算是请求开始 ？
——xhr.onloadstart事件触发的时候，也就是调用xhr.send()方法的时候。

因为xhr.open()只是创建了一个连接，但并没有真正开始数据的传输，而xhr.send()才是真正开始了数据的传输过程。只有调用了xhr.send()，才会触发xhr.onloadstart 。

那么什么时候才算是请求结束 ？
—— xhr.loadend事件触发的时候。

另外，还有2个需要注意：
  1. 可以在 send()之后再设置此xhr.timeout，但计时起始点仍为调用xhr.send()方法的时刻。
  2. 当xhr为一个sync同步请求时，xhr.timeout必须置为0，否则会抛错。

为什么会报错？ 继续

如何发一个同步请求
xhr默认发的是异步请求，也支持发同步请求。由xhr.open（）传入的async参数决定。
open(method, url [, async = true [, username = null [, password = null]]])
  ● method: 请求的方式，如GET/POST/HEADER等，这个参数不区分大小写
  ● url: 请求的地址，可以是相对地址如xxx.action，这个相对是相对于当前网页的url路径；也可以是绝对地址如http://www.example.com/xxx.action
  ● async: 默认值为true，即为异步请求，若async=false，则为同步请求


同步请求和异步请求只是阻塞和非阻塞的区别?  其他什么事件触发、参数设置应该是一样的?
事实证明并不是。

当xhr为同步请求时，有如下限制：
  ● xhr.timeout必须为0
  ● xhr.withCredentials必须为 false (xhr2 就不用设置了)
  ● xhr.responseType必须为""（注意置为"text"也不允许）
  ● 
若上面任何一个限制不满足，都会抛错，而对于异步请求，则没有这些参数设置上的限制。
之前说过页面中应该尽量避免使用sync同步请求，为什么呢？

因为我们无法设置请求超时时间（xhr.timeout为0，即不限时）。在不限制超时的情况下，有可能同步请求一直处于pending状态，服务端迟迟不返回响应，这样整个页面就会一直阻塞，无法响应用户的其他交互。
另外，标准中并没有提及同步请求时事件触发的限制。在 chrome中，当xhr为同步请求时，在xhr.readyState由2变成3时，并不会触发 onreadystatechange事件，xhr.upload.onprogress和 xhr.onprogress事件也不会触发。

???

如何获取上传、下载的进度
在上传或者下载比较大的文件时，实时显示当前的上传、下载进度是很普遍的需求。
我们可以通过onprogress事件来实时显示进度，默认情况下这个事件每50ms触发一次。
需要注意的是，上传过程和下载过程触发的是不同对象的onprogress事件：
  ● 上传触发的是xhr.upload对象的 onprogress事件
  ● 下载触发的是xhr对象的onprogress事件

```
xhr.onprogress = updateProgress;

xhr.upload.onprogress = updateProgress;
function updateProgress(event) {
    if (event.lengthComputable) {
      var completedPercent = event.loaded / event.total;
    }
}
```

可以发送什么类型的数据
void send(data);
xhr.send(data)的参数data可以是以下几种类型：
  ● ArrayBuffer
  ● Blob
  ● Document
  ● DOMString
  ● FormData
  ● null

如果是 GET/HEAD请求，send()方法一般不传参或传 null。不过即使真传入了参数，参数也最终被忽略，xhr.send(data)中的data会被置为 null.
xhr.send(data)中data参数的数据类型会影响请求头部content-type的默认值：
  ● 如果data是 Document 类型，同时也是HTML Document类型，则content-type默认值为text/html;charset=UTF-8;否则为application/xml;charset=UTF-8；
  ● 如果data是 DOMString 类型，content-type默认值为text/plain;charset=UTF-8；
  ● 如果data是 FormData 类型，content-type默认值为multipart/form-data; boundary=[xxx]
  ● 如果data是其他类型，则不会设置content-type的默认值
当然这些只是content-type的默认值，但如果用xhr.setRequestHeader()手动设置了中content-type的值，以上默认值就会被覆盖。

另外需要注意的是，若在断网状态下调用xhr.send(data)方法，则会抛错：Uncaught NetworkError: Failed to execute 'send' on 'XMLHttpRequest'。一旦程序抛出错误，如果不 catch 就无法继续执行后面的代码，所以调用 xhr.send(data)方法时，应该用 try-catch捕捉错误。
try{
    xhr.send(data)
  }catch(e) {
    //XXX...
  };


*xhr.withCredentials与 CORS 什么关系
在发同域请求时，浏览器会将cookie自动加在request header中。但大家是否遇到过这样的场景：在发送跨域请求时，cookie并没有自动加在request header中。

造成这个问题的原因是：在CORS标准中做了规定，默认情况下，浏览器在发送跨域请求时，不能发送任何认证信息，如"cookies"和"HTTP authentication schemes"。除非xhr.withCredentials为true.

所以根本原因是cookies也是一种认证信息，在跨域请求中，client端必须手动设置xhr.withCredentials=true，且server端也必须允许request能携带认证信息（即response header中包含Access-Control-Allow-Credentials:true），这样浏览器才会自动将cookie加在request header中。
另外，要特别注意一点，一旦跨域request能够携带认证信息，server端一定不能将Access-Control-Allow-Origin设置为*，而必须设置为请求页面的域名。

xhr相关事件

xhr相关事件有很多，下面是XMLHttpRequest的部分实现代码：
```
interface XMLHttpRequestEventTarget : EventTarget {
  // event handlers
  attribute EventHandler onloadstart;
  attribute EventHandler onprogress;
  attribute EventHandler onabort;
  attribute EventHandler onerror;
  attribute EventHandler onload;
  attribute EventHandler ontimeout;
  attribute EventHandler onloadend;
};

interface XMLHttpRequestUpload : XMLHttpRequestEventTarget {
};

interface XMLHttpRequest : XMLHttpRequestEventTarget {
  // event handler
  attribute EventHandler onreadystatechange;
  readonly attribute XMLHttpRequestUpload upload;
};
```

##### 从代码中我们可以看出：
  1. XMLHttpRequestEventTarget接口定义了7个事件：
      ○ onloadstart
      ○ onprogress
      ○ onabort
      ○ ontimeout
      ○ onerror
      ○ onload
      ○ onloadend
  2. 每一个XMLHttpRequest里面都有一个upload属性，而upload是一个XMLHttpRequestUpload对象
  3. XMLHttpRequest和XMLHttpRequestUpload都继承了同一个XMLHttpRequestEventTarget接口，所以xhr和xhr.upload都有第一条列举的7个事件
  4. onreadystatechange是XMLHttpRequest独有的事件

所以这么一看就很清晰了：
xhr一共有8个相关事件：7个XMLHttpRequestEventTarget事件+1个独有的onreadystatechange事件；而xhr.upload只有7个XMLHttpRequestEventTarget事件。

事件触发条件：
onreadystatechange 每当xhr.readyState改变时触发；但xhr.readyState由非0值变为0时不触发。
onloadstart 调用xhr.send()方法后立即触发，若xhr.send()未被调用则不会触发此事件。
onprogress xhr.upload.onprogress在上传阶段(即xhr.send()之后，xhr.readystate=2之前)触发，每50ms触发一次；xhr.onprogress在下载阶段（即xhr.readystate=3时）触发，每50ms触发一次。
onload 当请求成功完成时触发，此时xhr.readystate=4
onloadend 当请求结束（包括请求成功和请求失败）时触发
onabort 当调用xhr.abort()后触发
ontimeout xhr.timeout不等于0，由请求开始即onloadstart开始算起，当到达xhr.timeout所设置时间请求还未结束即onloadend，则触发此事件。
onerror 在请求过程中，若发生Network error则会触发此事件（若发生Network error时，上传还没有结束，则会先触发xhr.upload.onerror，再触发xhr.onerror；若发生Network error时，上传已经结束，则只会触发xhr.onerror）。注意，只有发生了网络层级别的异常才会触发此事件，对于应用层级别的异常，如响应返回的xhr.statusCode是4xx时，并不属于Network error，所以不会触发onerror事件，而是会触发onload事件。


*在哪个xhr事件中注册成功回调？

从上面介绍的事件中，可以知道若xhr请求成功，就会触发xhr.onreadystatechange和xhr.onload两个事件。 那么我们到底要将成功回调注册在哪个事件中呢？
是xhr.onload事件，因为xhr.onreadystatechange是每次xhr.readyState变化时都会触发，而不是xhr.readyState=4时才触发。
```
xhr.onload = function () {
    //如果请求成功
    if(xhr.status == 200){
        //do successCallback
    }
}
```

上面的示例代码是很常见的写法：先判断http状态码是否是200，如果是，则认为请求是成功的，接着执行成功回调。这样的判断是有坑儿的，比如当返回的http状态码不是200，而是201时，请求虽然也是成功的，但并没有执行成功回调逻辑。所以更靠谱的判断方法应该是：当http状态码为2xx或304时才认为成功。

```
xhr.onload = function () {
    //如果请求成功
    if((xhr.status >= 200 && xhr.status < 300) || xhr.status == 304){
        //do successCallback
    }
}
```
