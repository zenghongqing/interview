### 1. 常见的POST提交数据方式对应的content-type取值
1. application/x-www-form-urlencoded: 数据被编码为名称/值对，这是标准的编码格式
2. multipart/form-data: 数据被编码为一条消息，页上的每个控件对应消息的一个部分
3. application/json: 告诉服务端消息主体是序列化后的JSON字符串
4. text/xml
## webpack面试考点
### 1. HMR原理
如果我们使用了 webpack-dev-server，那么就是从 webpack-dev-server/bin/webpack-dev-server.js 启动的。webpack-dev-server包含三部分:
* webpack, 负责编译代码
* webpack-dev-server, 主要提供in-memory内存文件系统, 它会把webpack的outputFileSystem替换成inMemoryFileSystem, 并拦截全部的浏览器请求, 从这个文件系统中把结果取出来返回
* express 作为服务器
**总结webpack在服务端进行热更新的流程**
初始化阶段: 1 webpack-dev-server初始化的时候
* var compiler = webpack(options) // 创建webpack实例
* 监听compiler也就是webpack的done事件
* 创建express实例
* 创建WebpackDevMiddleware实例
* 设置express router, WebpackDevServer会作为express的一个中间件拦截请求
2 WebpackDevMiddleware在初始化时
* 创建一个MemoryFileSystem实例，替换掉webpack.outputFileSystem, 这样webpack编译出的文件其实都是在内存中, 而不是在磁盘上
```
function Server(compiler, options) {
  compiler.plugin('done', (stats) => {
    this._sendStats(this.sockets, stats.toJson(clientStats)); // 当完成编译的时候，就通过 websocket 发送给客户端一个消息（一个 `hash` 和 一个`ok`)
    this._stats = stats;
  });

  // Init express server
  const app = this.app = new express(); // eslint-disable-line

  // middleware for serving webpack bundle
  this.middleware = webpackDevMiddleware(compiler, options);
}

// 后面会有一行
app.use(this.middleware); // middleware会拦截所有请求，如果发现对应的请求是要请求 `dist` 中的文件，则会进行处理。
```
热更新阶段: 1、webpack监听文件变化，并完成编译 2、webpack-dev-server监听done事件, 并通过websocket向客户端发送消息 3、客户端经过处理后，请求新的JS模块代码 4、WebpackDevServer从MemoryFileSystem中取出代码并返回
**总结webpack在客户端进行热更新的流程**
1. 初始化的时候, client.js会启动一个socket和webpack-dev-server建立连接, 然后等待hash和ok消息
2. 当文件内容有改动时, 首先会收到webpack-dev-server发来的hash消息，得到新的hash值并保存起来
3. 然后会立刻接收到ok消息，表示现在可以加载最新代码了，于是进入reloadApp方法
4. reloadApp -> check()
5. check => hotDownloadManifest, 这里会下载一个本次热更新的manifest文件，url就是用上面存的 hash 拼接出来的，大概这样：8b52a72952cca784407e.hot-update.json，结果大概长这样：{"h":"8b52a72952cca784407e","c":{"0":true}}。这里仔细观察会发现，每一次取到的manifest中的hash 都是上一次 hash 消息的值，这样应该是为了保证顺序。
6. hotDownloadManifest 下载完配置文件后，可以看到其中有一个 h ，这个hash就是我们等会要取编译后的新代码的地址，在 hotEnsureUpdateChunk 方法中最终会通过 jsonp的方式把新的代码加载进来
7. 加载新的模块代码后，会更新modules tree 如installedModules的更新操作
8. 在hotApply中会执行我们的module.hot.accept注册的回调函数

### 2. Tree Shaking原理
webpack在tree-shaking中扮演的角色: 会进行无用导出的分析，把对应的export删除，但变量本身的声明并不会删除。删除代码的操作是uglify来做的。

### 3. webpack原理
流程为:
1. optimist模块命令行参数处理: 解析shell和config中的配置项, 用于激活webpack的加载项和插件
2. webpack初始化: 构建compiler对象、初始化webpak内部基础插件、初始化compiler的上下文、loader和file的输入输出环境
3. run() 编译的入口, compiler具体分为两个对象: 
* Compiler: 存放输入输出相关配置信息和编译器Parser对象
* Watching: 监听文件变化的一些处理方法
4. compile阶段, run触发compile, 接下来开始构建options中的模块, 构建compilation对象: 该对象负责组织整个编译过程, 包含了整个编译环节对应的方法；对象内部保留了compiler对象的引用，并存放了chunks和modules, 生成的assets以及最后用来生成js的template
5. 编译, compile中触发make事件并调用addEntry, 找到入口js文件, 进行下一步的模块绑定
6. 构建, _addModuleChain + addModuleDependencies方法: 解析入口js文件, 通过对应的工厂函数创建模块, 保存到compilation对象上, 然后对module进行build(buildModule+module.build方法), 包括调用loader处理源文件, 使用acorn生成AST并遍历AST, 遇到require等依赖时, 创建Dependency加入到依赖数组, module构建完毕，此时开始处理依赖的module, 异步的对依赖的module进行build，如果依赖中还有依赖，则循环处理其依赖
7. seal 封装构建结果, 逐次对每个module和chunk进行整理, 生成编译后的源码，合并、拆分, 每一个chunk对应一个入口文件, 开始处理最后生成的js
8. createChunkAssets() + 模板渲染: 所有的module, chunk仍然保存的是通过一个个require()聚合起来的代码，不同的dependencyTemplate如commonjs、AMD去render,  ModuleTemplate 是对所有模块进行一个代码生成, 通过调用module.source()来进行各种操作, 如__webpack_require()的格式
9. assets生成: 各模块进行 doBlock 后，把 module 的最终代码循环添加到 source 中。一个 source 对应着一个 asset 对象，该对象保存了单个文件的文件名( name )和最终代码( value )
### 4. 什么是bundle, 什么是chunk, 什么是module?
bundle是由webpack打包出来的文件, chunk是指webpack在进行模块的依赖分析时，代码分割出来的代码块；module是开发中的单个模块。
## JavaScript
1. 函数参数是对象的情况: 函数传参是传递对象指针的副本
```
function test(person) {
  person.age = 26
  person = {
    name: 'yyy',
    age: 30
  }

  return person
}
const p1 = {
  name: 'yck',
  age: 25
}
const p2 = test(p1)
console.log(p1) // -> {name: "yck", age: 26}
console.log(p2) // -> {name: "yyy", age: 30}
```
2. setTimeout、setInteval以及requestAnimationFrame区别
* setTimout本质上是多久后将回调函数推入执行队列，如果此时执行队列中还有其他任务，则不会立刻执行, 否则立刻执行, 当然可以通过代码去修正setTimeout，从而使定时器相对准确
* setInteval每隔一段时间执行一次回调函数, 但是因为它不仅不能保证在预期的时间执行任务, 而且存在执行累计问题
* requestAnimationFrame自带函数节流效果, 因为去的是系统时间，所以基本可以保证16.6毫秒内只执行一次(不掉帧的情况下),并且该函数的延时效果是精确的
3. instanceof原理
```
function myInstance(left, right) {
    let proto = right.prototype,
    left = left.__proto__
    while (true) {
        if (left === null || left === undefined)   return false
        if (left === proto) return true
        left = left.__proto__
    }
}
```
4. V8下的垃圾回收机制
V8实现了准确式GC, GC算法采用了分代式垃圾回收机制。V8将内存(堆)分为新生代和老生代两部分。
**新生代空间**: 用于存活较短的对象
分成两个空间: from空间与to空间
Scavenge GC算法: 当from空间占满时, 启动GC算法:
(1) 存活的对象从from space转移到to space
(2) 清空from space
(3) from space与to space 互换
(4) 完成一次新生代GC
**老生代空间**: 用于存活时间较长的对象
从新生代空间转移到老生代空间的条件: 经历一次以上Scavenge GC的对象和to space体积超过25%
**标记清除算法**: 标记存活的对象，未被标记的则被释放
* 增量标记: 小模块标记，在代码执行间隙, GC会影响性能
* 并发标记: 不阻塞js执行
**压缩算法**: 将内存中清楚后导致的碎片化对象往内存堆的一端移动, 解决内存的碎片化
5. Proxy相比于defineProperty的优势
* 数组变化也能监听到
* 不需要深度遍历监听
6. ES5和ES6继承的区别
ES5的继承实质上是先继承子类的实例对象, 然后再将父类的方法添加到this上(Parent.call(this))
ES6则是先创建父类的实例对象this, 然后再用子类的构造函数修改this。因为子类没有自己的对象, 所以必须先调用父类的super()方法, 否则新建实例报错。
7. 实现图片懒加载以及提高图片懒加载的效率
首先先自定义属性如:data-imgurl,存放着图片的路径，然后通过js判断界面滚动的位置/图片是否已加载, 未加载再去获取属性data-imgurl的值赋给src。节流
8. 静态资源加载和更新的策略
https://blog.csdn.net/zhangjs712/article/details/51166748
9. hash和history路由
这里的 hash 就是指 url 尾巴后的 # 号以及后面的字符。这里的 # 和 css 里的 # 是一个意思。hash 也 称作 锚点，本身是用来做页面定位的，她可以使对应 id 的元素显示在可视区域内。
由于 hash 值变化不会导致浏览器向服务器发出请求，而且 hash 改变会触发 hashchange 事件，浏览器的进后退也能对其进行控制，所以人们在 html5 的 history 出现前，基本都是使用 hash 来实现前端路由的。
已经有 hash 模式了，而且 hash 能兼容到IE8， history 只能兼容到 IE10，为什么还要搞个 history 呢？
首先，hash 本来是拿来做页面定位的，如果拿来做路由的话，原来的锚点功能就不能用了。其次，hash 的传参是基于 url 的，如果要传递复杂的数据，会有体积的限制，而 history 模式不仅可以在url里放参数，还可以将数据存放在一个特定的对象中。
history 模式改变 url 的方式会导致浏览器向服务器发送请求，这不是我们想看到的，我们需要在服务器端做处理：如果匹配不到任何静态资源，则应该始终返回同一个 html 页面。
10. 防抖和节流的区别
防抖: 触发高频事件后n秒内函数只会执行一次, 如果n秒内高频事件再次触发，则重新计算事件; 思路: 每次触发事件时都取消之前的延时调用方法
节流: 高频事件触发, 但在n秒内只会执行一次，所以会稀释函数的执行频率; 思路: 每次触发事件时都判断当前是否有等待执行的延时函数
11. 原型继承的种类并指出其优缺点
12. call、new和bind的模拟
13. 原型链图谱
14. 模拟Promise
15. 单例模式、装饰者模式、中介者模式、观察者模式、状态模式、职责链模式等了解
16. JS单线程的事件循环
17. Promise、yield以及async的关系(从原理上分析如: async是yield自执行函数的语法糖)
18. 算法: 快速排序、归并排序、插入排序、reduce的使用、大数相加、全排列、数组扁平化、函数柯里化等，主要围绕着递归方式执行
19. 事件捕获和事件冒泡的执行顺序是跟注册顺序一致的
20. this的指向、class的私有变量以及私有方法模拟问题
## 网络知识
1. Websocket原理
Websocket是HTML5下的一种新的持久化协议, 基于http。服务端可以主动push。
兼容: FLASH Socket, 长轮询: 定时发送ajax以及long poll: 发送 --> 有消息时再 response
优点: 支持双向通信, 实时性更强、更好的二进制支持和较少的控制开销。连接创建后, ws客户端、服务端进行数据交换时, 协议控制的数据包头部较小
2. 如何解决HTTP的无状态协议
Cookie: 第一次访问的时候给客户端发送一个Cookie，当客户端再次来的时候，拿着Cookie(通行证)，那么服务器就知道这个是”老用户“。
3. HTTP2.0
* 新的二进制格式
* 多路复用: 每一个request都是用做连接共享机制的
* header压缩: 使用encoder来减少需要传输的header大小, 通讯双方各自cache一份header fields表, 既避免了重复header的传输，又减小了需要传输的大小
* 服务端推送
4. Cookie和Session的区别
* cookie数据存放在客户的浏览器（客户端）上，session数据放在服务器上，但是服务端的session的实现对客户端的cookie有依赖关系的；
* cookie不是很安全，别人可以分析存放在本地的COOKIE并进行COOKIE欺骗，考虑到安全应当使用session；
* session会在一定时间内保存在服务器上。当访问增多，会比较占用你服务器的性能。考虑到减轻服务器性能方面，应当使用COOKIE；单个cookie在客户端的限制是3K，就是说一个站点在客户端存放的COOKIE不能超过3K；
5. TCP三次握手
建立连接前, 客户端和服务端需要通过握手来确认对方:
* 客户端发送syn(同步序列编号)请求, 进入syn_send状态, 等待确认
* 服务端接收并确认syn包后发送syn+ack, 进入syn_receive状态
* 客户端接收syn+ack包后, 发送ack包, 双方进入established
6. TCP三次握手
为什么 TCP 建立连接需要三次握手，明明两次就可以建立起连接?
因为防止出现失效的连接请求报文段被服务端接收的情况, 从而产生错误。
7. TCP四次挥手
第一次: 若客户端 A 认为数据发送完成，则它需要向服务端 B 发送连接释放请求
第二次: B 收到连接释放请求后，会告诉应用层要释放 TCP 链接。然后会发送 ACK 包，并进入 CLOSE_WAIT 状态，此时表明 A 到 B 的连接已经释放，不再接收 A 发的数据了。但是因为 TCP 连接是双向的，所以 B 仍旧可以发送数据给 A。
第三次: B 如果此时还有没发完的数据会继续发送，完毕后会向 A 发送连接释放请求，然后 B 便进入 LAST-ACK 状态。
第四次: A收到释放请求后, 向B发送确认应答, 此时A进入TIME-WAIT状态。该状态会持续 2MSL（最大段生存期，指报文段在网络中生存的时间，超时会被抛弃） 时间，若该时间段内没有 B 的重发请求的话，就进入 CLOSED 状态。当 B 收到确认应答后，也便进入 CLOSED 状态。
8. 跨域
* JSONP: 利用script标签不受跨域限制的特点, 缺点是只能支持get请求
```
function jsonp(url, jsonpCallback, success) {
    const script = document.createElement('script')
    script.src = url
    script.async = true
    script.type = 'text/javascript'
    window[jsonpCallback] = function (data) {
        success && success(data)
    }
    document.body.appendChild(script)
}
```
* CORS: Access-Control-Allow-Origin: *
* postMessage, 监听message事件
9. https流程
1.客户端：发送random1 + 支持的加密算法 + SSL Version等信息
2.服务端：发送random2 + 选择的加密算法A + 证书
3.客户端：验证证书 + 公钥加密的random3
4.服务端：解密random3，此时两端共有random1，random2，random3，使用这3个随机数通过加密算法计算对称密钥即可。

10. 将静态资源放在独立域名下有什么好处？
1 静态资源http请求不携带cookie，减少网站流量。2 动静分离有利于cdn。3 浏览器对于统一域名的下载线程有限（6个），优化下载速度。4 方便公司其他项目的使用。5 客户端缓存后 不同页面引用相同文件将直接从本地获取，无需下载
<br>
11. 用户在浏览器如何操作，会触发怎样的缓存策略:
(1) 打开网页，地址栏输入地址: 查找disk cache中是否匹配。如有则使用，如果没有则发送网络请求
(2) 普通刷新 (F5): 因为TAB并没有关闭, 因此memory cache是可用的, 会被优先使用(如果匹配的话)。其次才是disk cache
(3) 强制刷新 (Ctrl+F5): 浏览器不使用缓存, 因此发送的请求头部均带有Cache-control: no-cache(为了兼容，还带了 Pragma: no-cache), 服务器直接返回200和最新内容
<br>
12. http怎么解决持久连接？以及Content-Length的含义？
持久连接有两种类型：比较老的 HTTP/1.0+ "keep-alive" 连接，以及现代的 HTTP/1.1 "persistent" 连接;
http的协议中Content-Length首部告诉浏览器报文中实体主体的大小。这个大小是包含了内容编码的，比如对文件进行了gzip压缩，Content-Length就是压缩后的大小，而不是原始大小; 
除非使用了分块编码，否则Content-Length首部就是带有实体主体的报文必须使用的。使用Content-Length首部是为了能够检测出服务器崩溃而导致的报文截尾，并对共享持久连接的多个报文进行正确分段。
分块编码把「报文」分割成若干个大小已知的块，块之间是紧挨着发送的，这样就不需要在发送之前知道整个报文的大小了。（也意味着不需要写回Content-Length首部了）。
## web安全
CSRF: 跨站请求伪造
完成CSRF必要的三个条件:
(1) 用户已经登陆站点A，并本地记录了cookie
(2) 在用户没有登出站点A的情况下(cookie生效情况下),访问了恶意攻击者提供的引诱危险站点B(B站点要求访问站点A)
(3) 站点A没做任何CSRF防御
防御措施:
* Token验证
* Referer验证
* 验证码
XSS: 跨站脚本攻击: 反射型XSS和存储型XSS
* 反射型XSS
原理: 通过给别人发送带有恶意脚本代码参数的URL, 当URL被打开的时候, 特有的恶意代码参数会被HTML解析和执行
防御措施:
(1) web渲染所有的内容或渲染的数据必须来自服务端
(2) 尽量不要使用 eval, new Function()，document.write()，document.writeln()，window.setInterval()，window.setTimeout()，innerHTML，document.createElement() 等可执行字符串的方法
(3) 尽量不要从URL、document.referer以及document.forms 等这种 DOM API 中直接获取数据直接渲染
* 存储型XSS
原理: 一般存在于Form表单提交等交互功能如文章留言，提交文本信息等, 利用XSS的漏洞将内容经正常功能提交至数据库进行持久存储, 当前端页面获得服务端从数据库获取的代码时，恰好将其渲染
防御方法:
* 转义字符
* httpOnly Cookie 预防XSS攻击窃取用户cookie最有效的防御手段
* CSP 建立白名单 通常两种方式开启CSP: HTTP Header 中的 Content-Security-Policy和设置 meta 标签的方式
## CSS知识点
1. 为什么要初始化CSS样式？
因浏览器的兼容问题, 不同浏览器对有些标签的默认值是不一样的, 如果没对CSS初始化往往会出现浏览器之间的页面显示差异
2. 对BFC的理解?
BFC(块级格式化上下文)规定了内部的Block Box如何布局。定位方案:
* 内部的Box会在垂直方向上一个接一个放置
* Box垂直方向的距离由margin决定, 属于同一个BFC的两个相邻Box的margin会发生重叠
* 每个元素的margin box的左边与包含块border box的左边相接触
* BFC的区域不会与float box重叠
* BFC是页面上的一个隔离的独立容器，容器里面的子元素不会影响到外面的元素
* 计算BFC的高度时,浮动元素也会参与计算
满足下列条件之一就可以触发BFC
(1) 根元素 html
(2) float的值不为none
(3) overflow的值不为visible
(4) display的值为inline-block、table-cell、table-caption
(5) position的值为absolute或fixed
3. 垂直居中布局
(1) position + margin(transform)
(2) flex
(3) table
4. rem布局适配
## Vue
1. MVVM的含义
2. Diff算法的介绍
3. computed的内部实现
4. 数据双向绑定原理
5. nextTick原理
6. 介绍Vue生命周期各个阶段
7. 写一个简版的MVVM
## 浏览器渲染
1. window的onload事件和domcontentloaded谁先谁后?
DOMContentLoaded事件要在window.onload之前执行，当DOM树构建完成的时候就会执行DOMContentLoaded事件。当window.onload事件触发时，页面上所有的DOM，样式表，脚本，图片，flash都已经加载完成了。
2. 浏览器缓存
包括: diskStorage、memoryStorage、sessionStorage、localStorage、cookie
## Node.js
1. mongodb与mySQL的区别
mySQL是传统的关系型数据库，由数据库、表、记录三个层次组成, 但是在海量数据处理时效率会有所下降
mongodb是文档型数据库, 由数据库、集合文档三个层次组成, 呈现树状数据结构, 数据结构由键值对组成。但是mongodb是新型数据库, 不太稳定, 且不支持事务操作, 占用空间大
2. 介绍pm2
PM2是node进程管理工具, 可以利用它来简化很多node应用管理的繁琐任务, 如性能监控、自动重启、负载均衡等, 而且使用简单
3. Node的事件循环介绍
4. commonjs2的原理
5. 介绍Stream的flowing和pause模式
6. http协议头、状态码
7. 304过程
8. 命令行工具的写法(yargs)
9. 模块查找的顺序
10. nginx的配置: upstream的作用、静态资源的配置、强缓存的配置以及https的配置等