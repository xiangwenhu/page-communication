## 浏览器窗口间通信

补充一下，这里的通讯指遵守同源策略情况下。   

-------

为了吸引读者的兴趣，先把demo放到前面：   
下面有几个我自己写的演示**多页面通讯的demo**, 为了正常运行，请用最新的chrome浏览器打开。   
demo的源码地址[https://github.com/xiangwenhu/page-communication/tree/master/docs](https://github.com/xiangwenhu/page-communication/tree/master/docs)

* [首页](https://xiangwenhu.github.io/page-communication/)
* [setInterval + sessionStorage](https://xiangwenhu.github.io/page-communication/setInterval/index.html)
* [localStorage](https://xiangwenhu.github.io/page-communication/localStorage/index.html)
* [BroadcastChannel](https://xiangwenhu.github.io/page-communication/BroadcastChannel/index.html)
* [SharedWorker](https://xiangwenhu.github.io/page-communication/SharedWorker/index.html)

------

为什么会扯到这个话题，最初是源于听 https://y.qq.com/ QQ音乐，
*  播放器处于单独的一个页面 
*  当你在另外的一个页面搜索到你满意的歌曲的时候，点击播放或添加到播放队列
*  你会发现，播放器页面做出了响应的响应

这里我又联想到了商城的购物车的场景，体验确实有提升。   
刚开始，我怀疑的是Web Socket作妖，结果通过分析网络请求和看源码，并没有。 最后发现是localStore的storage事件作妖，哈哈。     

------
回归正题，其实在一般正常的知识储备的情况下，我们会想到哪些方案呢？

1. ### WebSocket
  这个没有太多解释，WebSocket 是 HTML5 开始提供的一种在单个 TCP 连接上进行全双工通讯的协议。当然是有代价的，需要服务器来支持。  
  js语言，现在比较成熟稳定当然是 [socket.io](https://github.com/socketio/socket.io)和[ws](https://github.com/websockets/ws). 也还有轻量级的[ClusterWS](https://github.com/ClusterWS/ClusterWS)。

你可以在[The WebSocket API (WebSockets)
](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)看到更多的关于Web Socket的信息。

2. ### 定时器 + 客户端存储

 定时器：setTimeout/setInterval/requestAnimationFrame    
 客户端存储： cookie/localStorage/sessionStorage/indexDB/chrome的FileSystem   

 定时器没啥好说的，关于客户端存储。
* cookie: 每次会带到服务端，并且能存的并不大，4kb?，记得不太清楚
* localStorage/sessionStorage 应该是5MB, sessionStorage关闭浏览器就和你说拜拜。
* indexDB 这玩意就强大了，不过读取都是异步的，还能存 Blob文件，真的是很high。
* chrome的FileSystem ,[Filesystem & FileWriter API](https://caniuse.com/#search=fileSystem),主要是chrome和opera支持。这玩意就是文件系统。



3.  ### postMessage
[Cross-document messaging](https://caniuse.com/#search=postMessage) 这玩意的支持率98.9%。 好像还能发送文件，哈哈，强大。   
不过仔细一看 window.postMessage()，就注定了你首先得拿到window这个对象。 也注定他使用的限制， 两个窗体必须建立起联系。 常见建立联系的方式：
* window.open
* window.opener
* iframe

**提到上面的window.open,  open后你能获得被打开窗体的句柄，当然也可以直接操作窗体了。**

-------------------

到这里，我觉得一般的前端人员能想到的比较正经的方案应该是上面三种啦。   
当然，我们接下来说说可能不是那么常见的另外三种方式。


1. ### StorageEvent
Page 1
```js
localStorage.setItem('message',JSON.stringify({
    message: '消息'，
    from: 'Page 1',
    date: Date.now()
}))
```

Page 2
```js
window.addEventListener("storage", function(e) {
    console.log(e.key, e.newValue, e.oldValue)
});
```
如上， Page 1设置消息， Page 2注册storage事件，就能监听到数据的变化啦。


上面的e就是[StorageEvent](https://developer.mozilla.org/en-US/docs/Web/API/StorageEvent),有下面特有的属性（都是只读）：
* key ：代表属性名发生变化.当被clear()方法清除之后所有属性名变为null
* newValue：新添加进的值.当被clear()方法执行过或者键名已被删除时值为null
* oldValue：原始值.而被clear()方法执行过，或在设置新值之前并没有设置初始值时则返回null
* storageArea：被操作的storage对象
* url：key发生改变的对象所在文档的URL地址


5. ### Broadcast Channel
这玩意主要就是给多窗口用的，Service Woker也可以使用。 firefox,chrome, Opera均支持，有时候真的是很讨厌Safari，浏览器支持75%左右。

使用起来也很简单, 创建BroadcastChannel, 然后监听事件。 只需要注意一点，渠道名称一致就可以。   
Page 1
```js
    var channel = new BroadcastChannel("channel-BroadcastChannel");
    channel.postMessage('Hello, BroadcastChannel!')
```
Page 2
```js
    var channel = new BroadcastChannel("channel-BroadcastChannel");
    channel.addEventListener("message", function(ev) {
        console.log(ev.data)
    });
```

6. ### SharedWorker
这是Web Worker之后出来的共享的Worker，不通页面可以共享这个Worker。  
MDN这里给了一个比较完整的例子[simple-shared-worker](https://github.com/mdn/simple-shared-worker)。   

这里来个插曲，Safari有几个版本支持这个特性，后来又不支持啦，还是你Safari，真是6。

虽然，SharedWorker本身的资源是共享的，但是要想达到多页面的互相通讯，那还是要做一些手脚的。
先看看MDN给出的例子的ShareWoker本身的代码：
```js
onconnect = function(e) {
  var port = e.ports[0];

  port.onmessage = function(e) {
    var workerResult = 'Result: ' + (e.data[0] * e.data[1]);
    port.postMessage(workerResult);
  }

}
```
上面的代码其实很简单，port是关键，这个port就是和各个页面通讯的主宰者，既然SharedWorker资源是共享的，那好办，把port存起来就是啦。   
看一下，如下改造的代码：    
SharedWorker就成为一个纯粹的订阅发布者啦，哈哈。
```js
var portList = [];

onconnect = function(e) {
  var port = e.ports[0];
  ensurePorts(port);
  port.onmessage = function(e) {
    var data = e.data;
    disptach(port, data);
  };
  port.start();
};

function ensurePorts(port) {
  if (portList.indexOf(port) < 0) {
    portList.push(port);
  }
}

function disptach(selfPort, data) {
  portList
    .filter(port => selfPort !== port)
    .forEach(port => port.postMessage(data));
}

```


Broadcast
>[MDN Web Docs - Broadcast Channel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel)    
[BroadcastChannel | Can I Use](https://caniuse.com/#search=BroadcastChannel)    
[broadcast-channel](https://github.com/pubkey/broadcast-channel)    
 BroadcastChannel that works in New Browsers, Old Browsers, WebWorkers and NodeJ   
--------

StorageEvent   
>[StorageEvent](https://developer.mozilla.org/en-US/docs/Web/API/StorageEvent)  

------

SharedWorker

>[SharedWorker](https://developer.mozilla.org/en-US/docs/Web/API/SharedWorker)    
[simple-shared-worker](https://github.com/mdn/simple-shared-worker/blob/gh-pages/worker.js)     
[SharedWorker | Can I Use](https://caniuse.com/#search=SharedWorker)   
[共享线程 SharedWorker](https://blog.csdn.net/qq_38177681/article/details/82048895)   
[feature-shared-web-workers](https://webkit.org/status/#feature-shared-web-workers) 
------


其他
>[两个浏览器窗口间通信总结](https://segmentfault.com/a/1190000016927268)      
