### Node介绍
Node.js 是一个基于 Chrome V8 引擎的 JavaScript 运行环境。 
Node.js 使用了一个事件驱动、非阻塞式 I/O 的模型，使其轻量又高效。
特点:
* 异步I/O
* 事件与回调函数
* 单线程: 不用像多线程编程那样处处在意状态的同步问题，这里没有死锁的存在，也没有线程上下文交换带来的性能上的开销，单线程的弱点: (1) 无法利用多核CPU (2) 错误会引起整个应用退出，应用的健壮性值得考验 (3) 大量计算占用CPU导致无法继续调用异步I/O
* 跨平台: 兼容windows和*nix平台得益于Node在架构层面的改动，它在操作系统与Node上层模块系统之间构建了一层平台层架构libuv。
#### Node的应用场景
* I/O密集型: I/O密集型的优势主要在于Nod利用事件循环的处理能力，而不是启动每一个线程为每一个请求服务，资源占用极少
* 是否不擅长CPU密集型业务: 由于单线程的原因，如果有长时间运行的计算(如大循环)，将导致CPU时间片不能释放，使得后续I/O无法发起。但是适当调整和分解大型运算任务为多个小任务，使得运算能适时释放，不阻塞I/O调用的发起，这样既可以享受到并行异步I/O的好处，又能充分利用CPU；Node虽然没有提供多线程用于计算支持，但是还是有以下两个方式充分利用CPU:
(1) Node可以通过编写C/C++扩展的方式更高效地利用CPU，将一些V8不能做到性能极致的地方通过C/C++来实现
(2) 通过子进程的方式，将一部分Node进程当做常驻服务进程用于计算，然后利用进程间的消息来传递结果，将计算I/O分离，这样能充分利用多CPU

### 模块机制
Node借鉴CommonJS的Modules规范实现了一套非常易用的模块系统，NPM对Packages规范的完好支持使得Node应用在开发过程中事半功倍;
#### Node的模块实现
在Node中引入模块，需要经历如下三个阶段:
(1) 路径分析
(2) 文件定位
(3) 编译执行
在Node中模块分为两种: Node提供的核心模块以及用户编写的文件模块:
* 核心模块部分在Node源代码编译的编译过程中，编译进了二进制执行文件。在Node进程启动时，部分模块编译进内存中，所以这部分核心模块引入时，文件定位和编译执行这两步可以省略掉，并且在路径分析中优先判断，所以它的加载速度是最快的
* 文件模块则是在运行状态时动态加载，需要完整的路径分析、文件定位、编译执行过程，速度比核心模块慢
#### 优先从缓存加载
不论是核心模块还是文件模块, require方法对相同模块的第二次加载都一律采用缓存优先的方式，这是第一优先级的。
模块路径的查找规则:
* 当前文件目录下的node_modules目录
* 父目录下的node_modules目录
* 沿路径向上逐级递归，直到根目录下的node_modules目录

#### 文件扩展名分析
CommonJS模块规范也允许在标识符中不包含文件扩展名，这种情况下，Node会按照.js、.json、.node的顺序补足扩展名，依次尝试; 如果是.node或者.json文件在传递给require()的标识中带上扩展名，会加快一点速度；另外如果同步配合缓存，可以大幅度缓解Node单线程中阻塞式调用的缺陷

#### 目录分析和包
在分析标识符的过程中，require通过分析文件扩展名之后，可能没有找到对应文件，但是却得到一个目录，这在引入自定义模块和逐个模块路径进行查找时经常会出现，此时Node会将目录当成一个包来处理;
在这个过程中Node对CommonJS包规范进行了一定程度的支持。首先Node会在当前目录下查找package.json，通过JSON.parse解析出包描述对象，从中取出main属性指定的文件名进行定位。如果缺少扩展名，将会进入扩展名分期的步骤。

#### 模块编译
定位到具体的文件后，Node会新建一个模块对象，然后根据路径载入并编译。
在编译过程中，Node对获取的JS文件内容进行头尾包装
```
(function (exports, require, module, __filename, __dirname) {})()
```
这样每个模块模块之间都进行了作用域隔离；内建核心模块的优势在于: 它们本身由C/C++编写，性能上优于脚本语言；其次，在进行文件编译时，它们被编译成二进制文件。一旦Node开始执行，它们被直接加载进内存中，无需再次做标识定位、文件定位、编译等过程，直接就执行

### npm机制

#### 安装查询命令
```
npm insatll packageName[@next | @versionNumber]
```
在node_modules中没有指定模块时安装.(不检查~/.npm目录)
```
npm install packageName --f | -- force
```
一个模块不管是否安装过，npm都要强制重新安装
```
npm update packageName
```
如果远程版本比较新、或者本地版本不存在时安装
* npm查询服务: npm通过registry的查询服务，从而知道每个模块的最新版本，可以通过 npm view packageName [version] 查询对映模块的信息

#### npm 安装机制
输入npm install命令并敲下回车，会经历如下阶段:
(1) 执行工程自身preinstall: 当前npm工程如果定义了preinstall钩子此时会执行
(2) 确定首层依赖模块: 首先要做的是确认工程中的首层依赖，也就是dependencies和devDependencies属性中直接指定的模块，工程本身就是整棵依赖树的根节点，每个首层依赖模块都是根节点下面的一颗子树，npm会开启多进程从每个首层依赖模块开始逐步寻找更深层级的节点;如果查询node_modules目录之中已经存在指定模块，那么不再重新安装
(3) 获取模块
获取模块是一个递归的过程，分为以下几步:
* 获取模块信息: 在下载一个模块前，首先要确定其版本，这是因为package.json中往往是语义化版本，此时如果版本描述文件（npm-shrinkwrap.json 或 package-lock.json中有该模块信息直接拿走即可，如果没有则从仓库获取。如packaeg.json 中某个包的版本是 ^1.1.0，npm 就会去仓库中获取符合 1.x.x 形式的最新版本。
* 获取模块实体: 上一步取到的模块的压缩包地址(resolved字段)，npm会用此地址检查本地缓存，缓存中有就直接拿，如果没有则从仓库下载
* 查找弄块依赖: 有就回到第一步，没有就结束
(4) 模块扁平化(dedupe)
从npm3版本开始默认加入了一个dedupe的过程，它会遍历所有节点，逐个将模块放在根节点下面，也就是node_modules的第一层。当发现有重复模块时，则将其丢弃。这里需要对重复模块进行一个定义，指的是模块名相同且semver兼容。每个 semver 都对应一段版本允许范围，如果两个模块的版本允许范围存在交集，那么就可以得到一个兼容版本，而不必版本号完全一致，这可以使更多冗余模块在 dedupe 过程中被去掉。
(5) 安装模块
这一步将会更新工程中的 node_modules，并执行模块中的生命周期函数（按照 preinstall、install、postinstall 的顺序）
(6) 执行工程自身生命周期
当前 npm 工程如果定义了钩子此时会被执行（按照 install、postinstall、prepublish、prepare 的顺序）。
最后一步是生成或更新版本描述文件，npm install 过程完成。

### Node的异步I/O
Nginx具备面向客户端管理连接的强大能力，但它的背后依然受限于各种同步方式的编程语言; 但Node却是全方位的，即可以作为服务端去处理客户端带来的大量并发请求，也能作为客户端向网络中的各个应用进行并发请求。
#### 事件循环
在进程启动时，Node会创建一个类似于while(true)的循环，每执行一次循环体的过程称之为Tick，每个Tick的过程就是看是否有事件待处理，如果有，就取出事件及其相关的回调函数。如果存在关联的回调函数，就执行它们。然后进入下一个循环，如果不再有事件处理就退出进程。
#### Node的I/O调用到回调函数执行中间发生了什么?
* 以fs.open()方法为例，从JS调用Node的核心模块，核心模块调用C++内建模块，内建模块通过libuv进行系统调用，这是Node的经典调用方式。这里libuv作为封装层，实质上是调用uv_fs_open()方法，在uv_fs_open()调用的过程中，我们创建了一个FSReqWrap请求对象，对象包装完后将其推入线程池中等待执行，至此JS调用立即返回，由JS层面发起的异步调用的第一阶段结束，JS线程可以继续执行当前任务的后续操作;
* 执行回调: 线程池中的I/O调用完毕后，会将获取的结果存储在req->result属性上，然后调用PostQueuedCompletionStatus()通知IOCP，告知当前对象操作已完成，这个过程中还动用了事件循环的I/O观察者，在每次Tick的执行中，它会调用相关的GetQueuedCompletionStatus()方法检查线程池中是否有执行完的请求，存在则会将请求对象加入到I/O观察者队列中，将其当做事件处理。I/O观察者回调函数的行为就是取出请求对象的result作为参数、取出oncomplete_sym属性作为方法，然后调用执行，以此达到调用JS中传入的回调函数的目的.。

#### 非I/O的异步API
* setTimeout()或者setInteval()创建的定时器会被插入到定时器观察者内部的一个红黑树中，每次Tick执行时会从该红黑树中迭代取出定时器对象，检查是否超时，如果超时就形成一个事件，它的回调函数立即调用。
* 调用process.nextTick(),只会将回调函数放入队列,在下一个Tick时取出执行，定时器采用红黑树的操作时间复杂度为O(lgn)，nextTick的时间复杂度为O(1)。
* process.nextTick()中的回调函数的优先级要高于setimmediate()，因为process.nextTick()处于idle观察者，setImmediate()处于check观察者，idle优先于I/O观察者，I/O观察者优先于check观察者
* 在具体实现上,process.nextTick()的回调函数保存在一个数组中, setImmediate()的结果则是保存在链表中。在行为上,process.nextTick()在每轮循环中会将数组中的回调函数全部执行完。而setImmediate()在每轮循环中执行链表中的一个回调函数
```
// timeout_vs_immediate.jssetTimeout(function timeout () {
  console.log('timeout');
},0);

setImmediate(function immediate () {
  console.log('immediate');
});
```
答案是不确定(由于setTimeout第二个参数默认为0，但是加上node做不到真正的0ms，最少也需要1s；所以实际执行进入事件循环后，如果没到1ms，那么timers阶段就会跳过进入check阶段，所以执行顺序不确定)

### 内存控制
Node中通过JavaScript使用内存时就发现只能使用部分内存(64位系统下约为1.4GB，32位系统下约为0.7GB)，在这样的限制下，将会导致Node无法操作大内存对象。

#### V8的垃圾回收机制
(1) V8的垃圾回收算法
V8的垃圾回收策略主要基于分代式垃圾回收机制，现代的垃圾回收算法中按对象的存活时间将内存的垃圾回收进行不同的分代，然后对不同分代的内存施以更高效的算法。
(2) V8的内存分代
在V8中，主要将内存分为新生代和老生代。新生代中的对象为存活时间较短的对象，老生代中的对象为存活时间较长或常驻内存的对象; V8使用的内存没办法根据使用情况自动扩充，当内存分配过程中超过极限值时，就会引进进程出错。新生代内存的最大值在64位系统和32位系统上分别为32M和64M，V8堆内存的最大值在64位系统上为1464MB，32位系统上则为732MB。
(3) Scavenge算法
新生代中的对象主要通过Scavenge算法进行垃圾回收，Scavenge算法主要采用Cheney算法，它是一种采用复制的方式实现的垃圾回收算法，将对内存一分为二，每一部分空间称为semispace，一个处于使用中(From空间)，一个处于闲置状态(To空间)。当我们分配对象时，先在From空间中进行分配，当开始进行垃圾回收时，会检查From空间中的存活对象，这些存活对象将被复制到To空间中，而非存活对象占用的空间将会被释放。完成复制后，From空间和To空间的角色发生对换;
* Scavenge算法的缺点就是只能使用堆内存的一半，这是由划分空间和复制机制所决定的
在分代式垃圾回收的前提下，From空间中的存活对象复制To空间之前进行检查，根据该对象的内存地址来判断这个对象是否经历了一次Scavenge回收，如果是就将From空间复制到老生代空间中，否则复制到To空间; 另一个判断条件是根据To空间的内存占比如果超过25%，则将该对象直接晋升到老生代空间中。

(4) Mark-Sweep和Mark-Compact
Mark-Sweep(标记清除)最大的问题是在进行一次标记清除回收后，内存空间会出现不连续的状态,这种内存碎片对后续的内存分配造成问题，因为可能需要分配一个大对象的时候。，为了解决该问题,提出Mark-Compact(标记整理)，对象在标记为死亡后，在整理的过程中，将活着的对象往一端移动，移动完成后清理掉边界外的内存。V8主要使用Mark-Sweep，在空间不足以对从新生代中晋升过来的对象进行分配时才使用Mark-Compact。
#### 内存泄露
原因:
(1) 缓存: 一旦一个对象被当做缓存来使用，那就意味着它将会常驻在老生代中，缓存中存储的键越多，长期存活的对象也就越多，这将导致垃圾回收在进行扫描和整理时，对这些对象做无用功，解决方案: 缓存限制策略
(2) 队列消费不及时, 解决方案: 监控队列的长度，一旦堆积，应当通过监控系统产生报警并通知相关人员，另一个方案是任意异步调用都应包含超时机制；
(3) 作用域未释放

### Buffer
Buffer主要用于操作字节，它所占用的内存不是通过V8分配的，属于堆外内存， 在Node的C++层面实现内存的申请的，因为处理大量的字节数据不能采用需要一点内存就向操作系统申请一点内存的方式，可能会造成大量的内存申请的系统调用，对操作系统有一定的压力。
* Node以8KB为界限来区分Buffer是大对象还是小对象，这个8KB的值也就是每个slab的大小值
(1) 分配小对象: 初次分配一个全新的slab单元，当前Buffer对象的parent属性指向该slab，并记录是从slab的哪个位置开始使用的， slab对象自身也记录被使用了多少字节，处于partial；当再次创建一个Buffer对象时，构造过程中会判断这个slab的剩余空间是否足够，如果足够，使用剩余空间，并更新slab的分配状态，如果不足，将会构造新的slab，原slab的剩余空间会造成浪费。
(2) 分配大Buffer对象: 如果需要分配大于8KB的Buffer对象，将会直接分配一个SlowBuffer对象作为slab单元，这个slab单元将会被这个大Buffer对象独占。

#### Buffer的拼接
(1) setEncoding()与string_decoder()
* 可读流rs的setEncoding方法作用是让data事件中传递的不再是一个Buffer对象，而是编码后的字符串;
* StringDecoder在得到编码后，知道宽字节字符串在UTF-8编码下以3个字节的方式存储，第一次write时，只会写入3的倍数个字节，剩余的字节会与第二次write时的字节拼接，再用3的整数倍字节转码，但是它的缺陷只能处理UTF-8、Base64和UCS-2、UTF-16LE这三种编码
(2) 正确的拼接方法
```
var chunks = []
var size = 0
res.on('data', function (chunk) {
  chunks.push(chunk)
  size += chunk.length
})
res.on('end', function () {
  var buf = Buffer.concat(chunks, size)
  var str = iconv.decode(buf, 'utf8)
  console.log(str)
})

Buffer.concat = function (list, length) {
	if (!Array.isArray(list)) {
		throw new Error('Usage: Buffer.concat(list, length)')
	}
	if (list.length === 0) {
		return new Buffer(0)
	} else if (list.length === 1) {
		return list[0]
	}
	if (typeof length !== 'number') {
		length = 0
		for (var i = 0; i < list.length; i++) {
			var buf = list[i]
			length += buf.length
		}
	}
	var buffer = new Buffer(length)
	var pos = 0
	for (var i = 0; i < list.length; i++) {
		var buf = list[i]
		buf.copy(buffer, pos)
		pos += buf.length
	}
	return buffer
}
```
(3) fs.createReadStream内部采用fs.read()实现，将会对磁盘的系统调用，对于大文件而言，highWaterMark的大小会决定触发系统调用和data事件的次数, highWaterMark值的大小与读取速度成正比

### 网络编程
TCP针对网络中的小数据包使用Nagle算法优化。如果每次发送一个字节的内容而不优化，网络中将充满只有极少数有效数据的数据包，将十分浪费网络资源。Nagle算法针对这种情况，要求缓冲的数据达到一定的数量或者一定时间后才将其发出，以此来优化网络。尽管在网络的一端调用write()会触发另一端的data事件，但是并不意味着每次write()都会触发一次data事件，在关闭掉Nagle算法后，接收端会将接受到的多个小数据包合并，然后只触发一次data事件。
#### http模块
* HTTP服务和TCP服务模型有区别的地方在于，在开启keep-alive后，一个TCP会话可以用于多次请求和响应。TCP服务以connection为单位进行服务，HTTP服务以request为单位进行服务。http模块将连接所用套接字的读写抽象为ServerRequest和ServerResponse对象，它们分别对应请求和响应操作。在请求产生的过程中，http模块拿到连接中传来的数据，调用二进制模块http_parser进行解析，在解析完请求报文的报头后，触发request事件，调用用户的业务逻辑。
* 响应报文头部设置可以调用setHeader进行多次设置，但只有调用writeHead后报文才会写入到连接中；一旦数据开始发送，writeHead()和setHeader()将不再生效
* HTTP服务事件
(1) connection事件: 在开始http请求和响应前，客户端与服务器需要建立底层TCP连接，这个连接可能因为开启了keep-alive，可以在多次请求响应之间使用；当这个连接建立时，服务器触发一次connection事件
(2) request事件: 当请求数据发送到服务器，解析出HTTP头部后将会触发该事件
(3) close事件: 停止接受新的连接，当已有的连接都断开时，触发该事件
(4) checkContinue: 某些客户端在发送较大的数据时，并不会直接发送数据，而是先发送一个头部带Expect: 100-continue的请求到服务器，服务器会触发checkContinue事件
(5) 当客户端发起CONNECT请求时触发，而发起CONNECT请求通常在HTTP代理时出现，如果不监听该事件，发起该请求的连接将会关闭
(6) upgrade事件: 客户端会在请求头部加上Upgrade字段，服务端会在接收到这样的请求时触发该事件
(7) clientError事件: 连接的客户端触发error事件时，这个错误会传递到服务器端，此时触发

#### webSocket服务
WebSocket最早是作为HTML5重要特性而出现的，最终在W3C和IETF的推动下，形成RFC6455规范，其优点:
* 客户端与服务端只建立一个TCP连接，可以使用更少的连接
* WebSocket服务器端可以推送数据到客户端，这远比HTTP请求响应模式更加灵活、高效
* 有更轻量级的协议头，减少数据传输量
WebSocket更接近于传输层协议，它并没有在HTTP的基础上模拟服务器端的推送，而是在TCP上定义独立的协议，主要分为两个部分: 握手和数据传输
(1) 握手
与普通的http请求协议头增加了如下协议头
```
Upgrade: webSocket,
Connection: Upgrade
```
该两个字段表示请求服务端升级协议为WebSocket，其中Sec-WebSocket-Key用于安全校验
```
Sec-WebSocket-Key: ***
```
Sec-WebSocket-Key是随机生成的Base64编码的字符串。客户端会校验响应返回的Sec-WebSocket-Accept字段，如果成功才会传输数据。

(2) 数据传输
握手顺利完成后，当前连接将不再进行HTTP的交互，而是开始WebSocket的数据帧协议，实现客户端与服务端的数据交换;
当客户端调用send()发送数据时，服务端触发onmessage()，当服务端调用send()发送数据时，客户端的onmessage()触发。当我们调用send()发送一条数据时，协议可能将这个数据封装为一帧或者多帧数据，然后逐帧发送; 为了安全考虑，客户端需要对发送的数据进行掩码处理，服务端一旦接受到无掩码帧(比如中间拦截破坏)，连接关闭。而服务端发送到客户端的数据帧则无需做掩码处理。
