作为前端开发，解决跨域问题应该是一个被熟练掌握的技能。而随着技术不断的更迭，针对跨域问题的解决也衍生出了多种解决方案。我们通常会根据项目的不同需要，而采取不同的方式。这篇文章，将详细总结跨域问题的相关知识点，以便在遇到相同问题的时候，能有一个清晰的解决思路。

## 背景

早期为了防止CSRF（跨域请求伪造）的攻击，浏览器引入了同源策略(SOP)来提高安全性。

> CSRF（Cross-site request forgery），跨站请求伪造，也被称为：one click attack/ session riding，缩写为：CSRF/XSRF。 —— [浅谈CSRF攻击方式](https://link.juejin.im/?target=http%3A%2F%2Fwww.cnblogs.com%2Fhyddd%2Farchive%2F2009%2F04%2F09%2F1432744.html)
>

而所谓"同源策略"，即同域名(domain或ip)、同端口、同协议的才能互相获取资源，而不能访问其他域的资源。在同源策略影响下，一个域名A的网页可以获取域名B下的脚本,css,图片等，但是不能发送Ajax请求，也不能操作Cookie、LocalStorage等数据。同源策略的存在，一方面提高了网站的安全性，但同时在面对前后端分离、模拟测试等场景时，也带来了一些麻烦，从而不得不寻求一些方法来突破限制，获取资源。

## JS跨域解决方案

这里所说的JS跨域，指的是在处理跨域请求的过程中，技术面会偏浏览器端较多一些，一般是利用浏览器的一些特性进行hack处理，从而避开同源策略的限制。

### 1、JSONP

由于同源策略不会阻止动态脚本的插入到文档中去，所以催生出了一种很常用的跨域方式： JSONP(JSON with Padding)。

原理说起来也很简单：

假设，我们源页面是在a.com,想要获取b.com的数据，我们可以动态插入来源于b.com的脚本:

```
script = document.createElement('script');
script.type = 'text/javascript';
script.src = 'http://www.b.com/getdata?callback=demo';
```

这里，我们利用动态脚本的src属性，变相地发送了一个 `http://www.b.com/getdata?callback=demo` 的GET请求。这时候，b.com 页面接受到这个请求时，如果没有JSONP,会正常返回json的数据结果，像这样：

```
{ msg: 'helloworld' }
```

而利用JSONP,服务端会接受这个callback参数，然后用这个参数值包装要返回的数据：

```
demo({msg: 'helloworld'});
```

这时候，如果a.com的页面上正好有一个demo的函数：

```
function demo(data) {
  console.log(data.msg);
}
```
当远程数据一返回的时候，随着动态脚本的执行，这个demo函数就会被执行。

到这里，你应该能明白这个技术为什么叫JSONP了吧？就是因为使用这种技术服务器会接受回调函数名作为请求参数，并将JSON数据填充进回调函数中去。

不过一般在实际开发的时候，我们一般会利用jQuery对JSONP的支持，而避免手写很多代码。从1.2版本开始，jQuery中加入了对JSONP的支持，可以使用$.getJSON方法来请求跨域数据：

```
//callback后面的?会由jQuery自动生成方法名
$.getJSON('http://www.b.com/getdata?callback=?', function(data) {
  console.log(data.msg);
});
```
还有一种更加常用的方法是，利用$.ajax方法，只要指定dataType为jsonp即可：

```
$.ajax({
  url: 'http://www.b.com/getdata?callback=?', //不指定回调名，可省略callback参数，会由jQuery自动生成
  dataType: 'jsonp',
  jsonpCallback: 'demo', //可省略
  success: function(data) {
    console.log(data.msg);
  }
});
```

虽然JSONP在跨域ajax请求方面有很强的能力，但是它也有一些缺陷。首先，它没有关于JSONP调用的错误处理，一旦回调函数调用失败，浏览器会以静默失败的方式处理。其次，它只支持GET请求，这是由于该技术本身的特性所决定的。因此，对于一些需要对安全性有要求的跨域请求，JSONP的使用需要谨慎一点了。

由于JSONP对于老浏览器兼容性方面比较良好，因此，对于那些对IE8以下仍然需要支持的网站来说，仍然被广泛应用。不过，针对高级浏览器，建议还是使用接下来会介绍的CORS方法。

### document.domain

目前，很多大型网站都会使用多个子域名，而浏览器的同源策略对于它们来说就有点过于严格了。如，来自www.a.com想要获取document.a.com中的数据。只要基础域名相同，便可以通过修改document.domain为基础域名的方式来进行通信，但是需要注意的是协议和端口也必须相同。

document.a.com中通过设置

```
document.domain = 'a.com';
```
www.a.com中：

```
document.domain = 'a.com';
var iframe = document.createElement('iframe');
iframe.src = 'http://document.a.com';
iframe.style.display = 'none';
document.body.appendChild(iframe);

iframe.onload = function() {
  var targetDocument = iframe.contentDocument || iframe.contentWindow.document;
  //可以操作targetDocument
}
```
最后，推荐一个使用iframe跨域的库 [`https://github.com/jpillora/xdomain`，感兴趣的可以去研究研究](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fjpillora%2Fxdomain%EF%BC%8C%E6%84%9F%E5%85%B4%E8%B6%A3%E7%9A%84%E5%8F%AF%E4%BB%A5%E5%8E%BB%E7%A0%94%E7%A9%B6%E7%A0%94%E7%A9%B6%E3%80%82)。


### window.name

window.name这个全局属性主要是用来获取和设置窗口名称的，但是通过结合iframe也可以跨域获取数据。我们知道，每个iframe都有包裹它的window对象，而这个window是最外层窗口的子对象。所以window.name属性就可以被共享。

下面这个简单的例子，展示了a.com域名下获取b.com域名下的数据：

```
var iframe = document.createElement('iframe');
iframe.style.display = 'none';
document.body.appendChild(iframe);

var isLoad = false;

//监听onload事件，获取window.name属性
iframe.onload = function() {
  if(isLoad) {
    //这里通过name属性获取到想要的数据了，通过parse来解析
    var data = JSON.parse(iframe.contentWindow.name);

    iframe.contentWindow.document.write('');
    iframe.contentWindow.close();
    document.body.removeChild(iframe);
  }else {
    //这里通过重定向url来获取域名b.com下的数据
    iframe.contentWindow.location = 'http://www.b.com/getdata.html';
    isLoad = true;
  }
}
```
b.com/getdata.html 中要存放的数据需要存储在 window.name 属性中：

```
<script>
var data = {msg: 'hello, world'};
window.name = JSON.stringify(data); // name属性只支持字符串，支持最大2MB的数据
</script>
```
还有一种iframe结合location.hash的方式，跟该方法十分类似：是通过检测iframe的src的hash属性来传递数据的。由于该方法相应速度较慢，这里就不做介绍了。

window.name+iframe的方法曾经被作为比JSONP更加安全的替代方案，然而对于托管敏感数据的现代Web应用程序来说，已经不推荐使用window.name来进行跨域消息传递了，而是推荐使用接下来介绍的`postMessage` API。


### window.postMessage

postMessage是HTML5新增在window对象上的方法，目的是为了解决在父子页面上通信的问题。该技术有个专有名词：跨文档消息(cross-document messaging)。利用postMessage的特性可以实现较为安全可信的跨域通信。

postMessage方法接受两个参数：

message: 要传递的对象，只支持字符串信息，因此如果需要发送对象，可以使用JSON.stringify和JSON.parse做处理
targetOrigin: 目标域，需要注意的是协议，端口和主机名必须与要发送的消息的窗口一致。如果不想限定域，可以使用通配符“*”,但是从安全上考虑，不推荐这样做。
下面介绍一个例子：

首先，先创建一个demo的html文件，我们这里采用的是iframe的跨域，当然也可以跨窗口。

```
<p>
  <button id="sendMsg">sendMsg</button>
</p>

<iframe id="receiveMsg" src="http://b.html">
</iframe>
```
然后，在sendMsg的按钮上绑定点击事件，触发 `postMessage `方法来发送信息给 iframe:

```
window.onload = function() {
  var receiveMsg = document.getElementById('receiveMsg').contentWindow; //获取在iframe中显示的窗口
  var sendBtn = document.getElementById('sendMsg');

  sendBtn.addEventListener('click', function(e) {
    e.preventDefault();
    receiveMsg.postMessage('Hello world', 'http://b.html');
  });
}
```

接着，你需要在iframe的绑定的页面源中监听 `message` 事件就能正常获取消息了。其中，`MessageEvent` 对象有三个重要属性：`data` 用于获取数据，`source` 用于获取发送消息的窗口对象，`origin` 用于获取发送消息的源。


```
window.onload = function() {
  var messageBox = document.getElementById('messageBox');

  window.addEventListener('message', function(e) {
    //do something
    //考虑安全性，需要判断一下信息来源
    if(e.origin !== 'http://xxxx') return;
    messageBox.innerHTML = e.data;
  });
}
```

总得来说，postMessage的使用十分简单，在处理一些和多页面通信、页面与iframe等消息通信的跨域问题时，有着很好的适用性。


## 服务器跨域

在实践过程中，一般我们喜欢让服务器来多做一些处理，从而尽可能让前端简化。这里将介绍两种常用的方法：反向代理和CORS。

### 反向代理

所谓反向代理服务器，它是代理服务器中的一种。客户端直接发送请求给代理服务器，然后代理服务器会根据客户端的请求，从真实的资源服务器中获取资源返回给客户端。所以反向代理就隐藏了真实的服务器。利用这种特性，我们可以通过将其他域名的资源映射成自己的域名来规避开跨域问题。

下面我将以node.js所写的服务器来做一个演示：

```
const http = require('http');

const server = http.createServer((req, res) => {

    const proxy_req = http.request({
        port: 8080,
        host: req.headers['host'],
        method: req.method,
        path: req.url,
        headers: req.headers
    });

    proxy_req.on('response', proxy_res => {
        proxy_res.on('data', data => {
            res.write(data, 'binary');
        });

        proxy_res.on('end', () => {
            res.end();
        });

        res.writeHead(proxy_res.statusCode, proxy_res.headers);
    });

    req.on('end', () => {
        proxy_req.end();
    });

    req.on('data', data => {
        proxy_req.write(data, 'binary');
    });
});

server.listen(80);
```

以上代码会将请求80端口的资源映射到8080端口上去。原理就是在监听到客户端请求后，启动一个代理服务器，然后获取代理服务器返回的结果，直接返回给客户端。

如果你使用的是express, 则代码量将更少，也很方便：

```
const express = require('express');
const request = require('request');
const app = express();

const proxyServer = 'localhost:8080';

app.use('/', (req, res) => {

  const url = proxyServer + req.url;

  req.pipe(request(url)).pipe(res);

});

app.listen(process.env.PORT || 80);
```

利用反向代理，你可以将任何请求委托给另外的服务器，从而避免在浏览器端进行跨域操作。不过你需要注意的是：不要使用bodyParser中间件，因为你需要直接将原始请求通过管道传输到外部服务器。

一般来说，如果你的生产环境上应用和API在同一台服务器上运行，就没有必要使用跨域了。 而在开发阶段采用这种反向代理，则更加方便我们前端开发和测试。

在使用反向代理上，你也可以借助 [node-http-proxy](https://link.juejin.im/?target=https%3A%2F%2Fgithub.com%2Fnodejitsu%2Fnode-http-proxy) 库来减少代码量。


### CORS

"跨域资源共享"（Cross-origin resource sharing）是W3C出的一个标准。兼容性方面可以支持IE8+（IE8和IE9需要使用XDomainRequest对象来支持CORS），所以现在CORS也已经成为主流的跨域解决方案。

CORS的核心思想是通过一系列新增的HTTP头信息来实现服务器和客户端之间的通信。所以，要支持CORS，服务端都需要做好相应的配置，这样，在保证安全性的同时也更方便了前端的开发。

浏览器会将CORS请求分为两类：简单请求和非简单请求：

#### 简单请求

在CORS标准中，会根据是否触发CORS preflight（预请求）来区分简单请求和非简单请求。

简单请求需要满足以下几个条件：

- 1.请求方法只允许：GET,HEAD,POST
- 2.对于请求头字段有严格的要求，一般情况下不会超过以下几个字段：

	- Accept
	- Accept-Language
	- Content-Language
	- Content-Type
- 3.当发起POST请求时，只允许 Content-Type 为 application/x-www-form-urlencoded，multipart/form-data,text/plain。

对于简单请求来说，服务器和客户端之间的通信只是进行简单的交换。

浏览器发送一个带有Orgin字段的HTTP请求头，用来表明请求来源。服务器的Access-Control-Allow-Origin响应头表明该服务器允许哪些源的访问，一旦不匹配，浏览器就会拒绝资源的访问。大部分情况，大家都喜欢将Access-Control-Allow-Origin设置为*,即任意外域都能访问该资源。但是，还是推荐做好访问控制，以保证安全性。


#### 非简单请求

对于非简单请求，情况就稍微复杂了点。在正式发送请求数据之前，浏览器会先发送一个带有'OPTIONS'方法的请求来确保该请求对于目标站点来说是安全的，这个请求也被称为”预请求“（preflight)。

浏览器和服务器之间具体的交互过程如图所示：

浏览器会在预检请求中，多发送两个字段Access-Control-Request-Method和Access-Control-Request-Headers,前者用于告知服务器实际请求所用的方法，后者用于告知服务器实际请求所携带的自定义请求首部字段。然后，服务器将根据请求头的信息来判断是否允许该请求。

针对非简单请求，服务器端可以设置几个相关字段:

1. Access-Control-Allow-Methods, 用来限制允许的方法名，
2. Access-Control-Allow-Header,用来限制允许的自定义字段名
3. Access-Control-Allow-Credentials，用来表明服务器是否允许credentials标志为true的场景。
4. Access-Control-Max-Age,用来表明预检请求响应的有效时间
5. Access-Control-Expose-Headers，用来指定服务器端允许的首部字段集合

另外，如果是在具体的实践过程中，调试OPTIONS请求可以使用

```
curl -X OPTIONS http://xxx.com
```

来进行查看相应头信息。也可以通过 `chrome://net-internals/#events` 来获取更加详细的网络请求信息。

#### 优化CORS

针对非简单请求来说，由于每个请求都会发送预请求，这就导致接口数据的返回会有所延迟，时间被加长。所以，在使用CORS的过程中，可以采用一些方案来优化请求，将非简单请求转换成简单请求，从而提高请求的速度。

**1.请求缓存**

可以在服务器端使用Access-Control-Max-Age来缓存预请求的结果。从而提高网站性能。但是需要注意的是，大部分浏览器不会允许缓存‘OPTIONS‘请求太长时间，如：火狐是24小时（86400s)，chromium是10分钟(600s)。

**2.针对GET请求**

对于GET请求，没必要使用Content-Type, 尽可能地保持GET请求是简单请求。这样就可以减少Header上所携带的字段。从安全性上考虑，所有的API调用应该尽可能使用https协议，而这样可以将一些授权认证信息（如token)直接放在url中去，而不必放在头部。

**3.针对POST请求**

对于POST请求，我们可以尽量使用FormData这种原生的格式

```
function sendQuery(url, postData) {
  let formData = new FormData();
  for(var key in postData) {
    formData.append(key, postData.key);
  }

  return fetch(url, {
    body: formData,
    headers: {
      'Accept': '*/*'
    },
    method: 'POST'
  });
}

sendQuery('http://www.xxx.com', {msg: 'hello'}).then(function(response) {
  //do something with response
});
```

#### 附带凭证信息的请求

CORS预请求会将用户的身份认证凭据排除在外，包括cookie、http-authentication报头等。如果需要支持用户凭证，需要在XHR的withCredentials属性设置为true，同时Access-control-allow-origin不能设置为*。在服务器端会利用响应报头Access-Control-Allow-Credentials来声明是否支持用户凭证。

同时，利用withCredentials这个属性，也可以检测浏览器是否支持CORS。下面创建一个带有兼容性处理的cors请求：

```
function createCORSRequest(method, url){
    var xhr = new XMLHttpRequest();
    if ("withCredentials" in xhr){
        xhr.open(method, url, true);
    } else if (typeof XDomainRequest != "undefined"){
        xhr = new XDomainRequest();
        xhr.open(method, url);
    } else {
        xhr = null;
    }
    return xhr;
}

var request = createCORSRequest("POST", "http://www.xxx.com");
if (request){
    request.onload = function(){
        //do something with request.responseText
    };
    request.send();
}
```

如果浏览器支持fetch，则使用它做跨域请求更加方便：

```
fetch('http://www.xxx.com', {
  method: 'POST',
  mode: 'cors',
  credentials: 'include' //接受凭证
}).then(function(response) {
  //do something with response
});
```

## 总结

以上介绍的这些跨域方法，可能有些已经很少使用了，但是这些方法在解决问题的思路上都有着一定的参考意义。所以当面对不可避免的跨域问题的时候，也希望这篇文章对你能有所帮助。


> 参考资料
>
> http://jirengu.com/app/album/54
>
> https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS
>
> http://damon.ghost.io/killing-cors-preflight-requests-on-a-react-spa/
>
> http://blog.teamtreehouse.com/cross-domain-messaging-with-postmessage
>
