title: 跨域
date: 2016-08-28 14:13:31
description: 
categories: 前端开发
tags:
  - CORS
toc:
feature:
---
跨域是指不同域之间的网络传输和数据交换,基于互联网的多样性和开放性进行限制的同源策略,即页面之间拥有相同的协议（protocol）,端口（如果指定,但IE例外）和主机.

事实上,广义的跨域应该包括基于不同域的资源之间的相互读写／收发.

## 1.请求
### 1.1 HTTP
[CORS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)对跨域服务端设置headers并制定跨域规则:
```
Access-Control-Allow-Origin
```
带验证信息的CORS:
```
Access-Control-Allow-Credentials
```
注意非简单请求的HTTP(CORS)都将先发送一个OPTIONS的预请求,以来查明这个跨站请求对于目的站点是不是安全可接受的.

简单请求:
```
请求以GET,HEAD或者POST(dataType:application/x-www-form-urlencoded,multipart/form-data,text/plain )的方法发出请求
```
PS:SSE的跨域也符合CORS规则,但SSE所有微软系列浏览器均不支持;@font-face字体的远程调用也是使用CORS解决..

适用范围:IE8+

### 1.2 Websocket
Websocket基于单个TCP连接上的全双工通道,可双向通信的协议,本身即支持跨域请求,但默认不支持断线重连,SSE默认支持.当然,Websocket同样也有基于安全连接的wss.

适用范围:IE10+
### 1.3 Fetch
这是新型浏览器接口标准用于替代XMLHttpRequest.
```
fetch(
    'url',
    {
        mode: 'cors'
    }
)
.then(function(response) {
    //do something
});
```
与XMLHttpRequest跨域一样,服务器需要支持预请求.

适用范围:IE全系列均不支持,Edge14+
## 2.存储
### 2.1 Cookie
跨站cookie共享方案:
```
var shareCookie = function(url,callback){
    var jsonp = document.createElement("script");
    jsonp.src = “http://test.com?_123”;
    script.onload = script.onreadystatechange = function() {
        if (!this.readyState || this.readyState == "loaded" || this.readyState == "complete") {
            callback();
        }
    };
}
shareCookie('a.com?getCookie',function(){
    console.log(shareCookie);//aSiteCookie
});

curl http://a.com?getCookie
var shareCookie = {cookie:aSiteCookie};
```
由于第三方cookie便于跨域跟踪用户等诸多隐私泄漏,IE较早阻止了第三方cookie,2012后Firefox和Safari陆续 默认阻止第三方cookie的策略.
P3P成为唯一解决第三方cookie的方案,即加入如下请求headers:
```
P3P: CP="CURa ADMa DEVa PSAo PSDo OUR BUS UNI PUR INT DEM STA PRE COM NAV OTC NOI DSP COR”
```

试用范围:均支持

### 2.2 localStorage(sessionStorage)
localStorage没有过期时间,并在BOM中加入全局监听事件'storage'用于localStorage任何改变与删除事件触发(用同浏览器同域且非隔离环境下的页面之间的交互),sessionStorage在该页面关闭时将清除.多数token验证均基于localStorage.

试用范围:IE8+

### 2.3 IndexedDB
对于单个数据库并无大小限制,但在单个IndexedDB数据库尽量小量结构化数据的存储(建议小于50M),IndexedDB仅异步API操作m目前多数浏览器均为实现同步API.

试用范围:IE10+
### 2.4 Web SQL Database
由于W3C已经放弃它的规范,多数浏览器的支持均基于SQLite的实现各自的API,因此在规范性上,已经失去标准统一性了,并不建议使用.

试用范围:IE全系列和Firefox均不支持

## 3.资源

**CSP**
通过目标地址配置headers:[Content-Security-Policy](https://developer.mozilla.org/zh-CN/docs/Web/Security/CSP/CSP_policy_directives)可定制目标网站页面的内容安全策略:
```
Content-Security-Policy:*-src * //*详细看API
```
chrome 45+起 默认开启禁止Inline JavaScript来防止XSS攻击.因此在chrome下配置依然无法屏蔽禁止内联script的返回并渲染:
```
Content-Security-Policy:default-src *
```
适用范围: IE10部分支持,IE12+
### 3.1 script(jsonp)
通过script请求一次带data的callback的全局回调函数.
```
window.test = function(data){
    console.log(data)
}
<script src="http://test.com?callback=test&otherParam=test">
curl http://test.com?callback=test&otherParam=test
test(data)//data为数据
```
jsonp的优点是全兼容,但只支持get请求,且占用污染全局变量.

适用范围:均支持
### 3.2 link(css with content)
通过Javascript载入一个css文件,并创建某个dom插入节点setInterval该节点的样式变化后,取得该css中的content数据.

同样只支持get请求,且污染目标页的DOM.

适用范围:均支持
### 3.3 img(exif)
创建一个img的节点设置src,利用img的onload触发获得带exif的图片数据.值得一提的是,测试访问目标页用户的网络状态,也可利用设置src与onload之间的时间差获得判断.

缺点不同后端语言并未有完整的exif现实库.

适用范围:均支持
### 3.4 flash
```
AS3：
    flash.system.Security.allowDomain("*");//http
    flash.system.Security.allowInsecureDomain("*");//https
HTML:
    <param name="allowScriptAccess" value="always" />
```
2015开始flash陆续遭到摒弃.

适用范围:基于flash客户端支持

### 3.5 frame / iframe

**如何Post提交数据**
```
var postData = function(url, data) {
    var tempForm = document.createElement("form"),
        tempFrame = document.createElement('iframe');
    tempForm.method = "post";
    tempForm.action = url;
    tempForm.target = 'tempFrame';
    for(var i in data){
        var hideInput = document.createElement("input");
        hideInput.type = "hidden";
        hideInput.name = i;
        hideInput.value = data[i];
        tempForm.appendChild(hideInput);
    }
    tempFrame.name = 'tempFrame';
    tempFrame.style.display = 'none';
    tempFrame.src = 'about:blank';
    tempFrame.onload = function () {
         setTimeout(function () {
              if(tempFrame.src==='about:blank') return;
              document.body.removeChild(tempFrame);
         },0);
    };
    document.body.appendChild(tempFrame);
    document.body.appendChild(tempForm);
    tempForm.submit();
    document.body.removeChild(tempForm);
}
```
**3.5.1 src的hash**

利用src的hash信息共享获得iframe父子页面之间的数据交换.缺点是信息量受URL(255字节)长度限制,且信息容易泄漏.

**3.5.2 postMessage**

postMessage可用于iframe父子之间的信息传递交互,并通过'message'事件来监听.另外需要补充的是worker也支持postMessage,适用范围IE8+,是目前较安全和推荐的跨域解决方案.

**3.5.3 document.domain**
在主域名相同只域名不同的情况下,设置document.domain为主域名,可解决子域名间的跨域问题:
```
document.domain = 'siteDomain.com'
```
**3.5.4 window.navigator**
在IE6-7一直存在的bug是window.navigator是可以共享访问的.


**3.5.6 window.name**
```
在页面A设置了window.name='data',并进行当前页url跳转后B页面在访问window.name可得到相同数据
```
利用该原理,创建一个iframe并将数据设置为window.name中,然后跳转至父页面的本域下页面或者'about:blank'的跨域继承,并监听iframe的onload触发获取window.name.

该方案的优点上全兼容,缺点是污染了全局变量.

参考引用:
* [1][https://fetch.spec.whatwg.org/](https://fetch.spec.whatwg.org/)
* [2][https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)
* [3][https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)