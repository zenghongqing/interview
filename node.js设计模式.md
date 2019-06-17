### Reactor模式简介
1. 应用程序通过向Event Demultiplexer(事件多路分解器)提交请求来生成新的I/O操作。应用程序还指定一个处理程序(Node.js中指回调函数)，当操作完成时将调用该处理程序。向Event Demultiplexer提交新请求是一种非阻塞调用，它立即将控制权返回给应用程序
2. 当一组I/O操作完成时，事件多路分解器将新的事件推入Event Queue(事件队列)
3. 此时，Event Loop遍历Event Queue的项目
4. 对于每个事件，调用相关联的处理程序
5. 处理程序只是应用程序代码的一部分，当它执行完成时将把控制权返回给Event Loop。但是在处理程序执行过程中可能会请求新的异步操作，从而导致新的操作会被插入Event Demultiplexer
6. 当Event Loop中的所有项目被处理完时，循环将再次阻塞Event Demultiplexer，当有新事件可用时，Event Demultiplexer将触发另一个周期

### Node.js基础设计模式
#### 回调(CSP)
* 解决Zalog
1. 延迟执行: 使用process.nextTick()来实现，它的作用是延迟一个函数的执行，直到下一次事件循环
2. 转为同步，使用同步API
* Node.js回调约定
1. 回调函数置尾
2. 暴露错误优先
3. 传播错误: 当传播错误时，应使用return语句，避免继续执行
4. 未捕获异常: 在异步回调中抛出异常将导致异常跳转到事件循环，并且不会传播到下一个回调，可以使用process.on('uncaughtException, err => {})监听
#### 观察者模式 (EventEmitter类)
```
const EventEmitter = require('events').EventEmitter
const instance = new EventEmitter()
instance.on(event, listener)
instance.once(event, listener)
instance.emit(event, [arg1, ...])
removeListener(event, listener)
```

### 异步控制流
* 顺序迭代
```
// 递归
function iterate (index) {
    if (index === tasks.length) {
        return finish()
    }
    const task = tasks[index]
    task(() => {
        iterate(++index)
    })
}
// iteration finish
function finish () {}
iterate(0)
```
* 并发执行
因为任务不是同时运行的，而是由底层的非阻塞API执行，并由事件循环交叉运行
```
// 并行运行一组异步任务，同时生成所有异步任务，然后计算它们的回调被调用的次数，等待所有异步任务完成
const tasks = [...]
let completed = 0
tasks.forEach(task => {
    task(() => {
        if (++completed === tasks.length) {
            finish()
        }
    })
})
function finish () {}
```
* 有限制的并行执行
为防止资源过度负载如资源耗尽，在web应用程序中，还可能创造可利用的拒绝服务( DoS)攻击的漏洞，所以最好限制可以同时运行的任务数
```
const tasks = [...]
let concurrency = 2, running = 0, completed = 0, index = 0
function next () {
    while (running < concurrency && index < tasks.length) {
        task = tasks[index++]
        task(() => {
            if (completed === tasks.length) {
                return finish()
            }
            completed++
            running--
        })
        running++
    }
}
next()
function finish () {}
```
* Node.js风格函数的promise化
```
module.exports.promisify = function (callbackBaseApi) {
    return function promisified (...args) {
        const arguments = [...args]
        return new Promise((resolve, reject) => {
            arguments.push((...argvs) => {
                if (err) return reject(err)
                if (argvs.length <= 2) {
                    resolve(result[1])
                } else {
                    resolve(args.slice(1))
                }
            })
            callbackbaseApi.apply(null, arguments)
        })
    }
}
```
thunk化
```
function readFileThunk (filename, options) {
    return function (callback) {
        fs.readFile(filename, options, callback)
    }
}
```

### 流
Node.js是以事件为基础，处理I/O操作最高效的方法就是实时处理，尽快的接收输入内容，并经过程序的处理尽快的输出结果。
Node.js中的每个流的实例都是由stream模块提供的四个基本抽象类之一实现的
* stream.Readable
* stream.Writable
* stream.Duplex
* stream.Transform
流支持两种模式:
* 二进制模式: 在该模式下，流中的数据是以块的形式存在，如缓存或字符串
* 对象模式: 在该模式下，流中的数据被看作一系列独立的对象
#### 可读流
一个可读流代表了一个数据源，在Node.js中，可以使用stream模块提供的Readable抽象类实现；从可读流中获取数据有两种模式: 非流动(non-flowing)模式和流动(flowing)模式
(1) 非流动模式
从可读流中读取数据的默认方式都是添加一个对于readable事件的监听器，在读取新的数据时进行通知。然后在一个循环中，读取所有的数据直到内部缓存被清空，通过read()方法来实现，该方法能同步读取缓存中的数据并返回一个Buffer或者String对象表示数据块
```
process.stdin.on('readable', () => {
    let chunk
    console.log('new data available')
    while((chunk = process.stdin.read()) !== null) {
        console.log(`Chunk read: (${chunk.length}) (${chunk.toString()})`)
    }
}).on('end', () => process.stdout.write('End of stream'))
```
当内部缓存中没有更多的数据可读时，read()方法会返回null，这时必须等待readable事件再次触发，告诉我们有新的数据可读或者等待end事件，告诉整个可读流结束。在二进制模式下，还可以在调用的read()方法时指定size的值，表示想要读取的数据大小。在实现网络协议或者解析特定的格式数据时非常有用。
(2) 流动模式
另一种从流中读取数据的方式是个data事件添加一个监听器:
```
process.stdin.on('data', chunk => {
    console.log('new data available')
    console.log(`Chunk read: (${chunk.length}) (${chunk.toString()})`)
}).on('end', () => process.stdout.write('End of stream'))
```
在stream2中，流动模式不是默认的工作模式，如果想要启动，需要显示的为data事件添加监听器或者调用resume()方法。可以调用pause()方法来实现临时阻止流触发data事件，将接收到的数据存到内部缓存中。调用pause()并不会导致流转换为非流动模式。
(3) 实现可读流
需要创建一个新的类继承stream.Readable的原型，具体的流实例必须提供对于_read()方法的实现: readable._read(size)，Readable类内部会调用_read()方法，紧接着调用push()方法将数据填充到缓存中readable.push(chunk)。
```
class RandomStream extends stream.Readable {
    constructor (options) {
        super(options)
    }
    _read (size) {
        const chunk = chance.toString()
        console.log(`Pushing chunk of size: ${chunk.length}`)
        this.push(chunk, 'utf8')
        if (chance.bool({likelihood: 5})) {
            this.push(null)
        }
    }
}
randomStream.on('readable', () => {
    let chunk
    if ((chunk = randomStream.read()) !== null) {
        console.log(`Chunk received: ${chunk.toString('utf8')}`)
    }
})
```
如果一次调用多次推送数据，则需要检查push()方法返回是否为false，表示内部缓存达到了highWaterMark的限制，此时应该停止将继续更多的数据添加到流中。
#### 可写流
可写流表示数据目的地，在Node.js中可以使用流模块提供的抽象类stream.Writable来实现。向可写流中写入数据采用
```
writable.write(chunk, [encoding], [callback])
// 不再将更多的数据写入流中, callback函数相当于为finish事件注册的监听器
writable.end(chunk, [encoding], [callback])
```
为防止数据写入流的速度比从流中读取速度快(导致数据在缓存内积聚越来越多),当内部缓存超过了highWaterMarkW的限制时，write()方法会返回false，此时停止数据继续写入。当清空缓存后，触发drain事件，可以重新执行写操作,成为背压机制(back-pressure)。
```
class ToFileStream extends stream.Writable {
    constructor () {
        // 指定流以对象模式
        super({objectMode: true})
    }
    _write (chunk, encoding, callback) {
        mkdirp(path.dirname(chunk.path), err => {
            if (err) {
                return callback(err)
            }
            fs.writeFile(chunk.path, chunk.content, callback)
        })
    }
}
const tfs = new ToFileStream()
tfs.write({path: 'file.txt', content: '你好'})
```
#### 双向流(Duplex stream)
变换流是一种特殊的双向流，用来处理数据的转换。
```

class ReplaceStream extends stream.Transform {
    constructor (searchString, replaceString) {
        super()
        this.searchString = searchString
        this.replaceString = replaceString
        this.tailPiece = ''
    }
    _transform (chunk, encoding, callback) {
        const pieces = (this.tailPiece + chunk).split(this.searchString)
        const lastPieces = pieces[pieces.length - 1]
        const tailPieceLen = this.searchString.length - 1
        
        this.tailPiece = lastPieces.slice(tailPieceLen)
        pieces[pieces.length - 1] = lastPieces.slice(0, -tailPieceLen)
        this.push(pieces.join(this.replaceString))
        callback()
    }
    _flush (callback) {
        this.push(this.tailPiece)
        callback()
    }
}
const rs = new ReplaceStream('World', 'Node.js')
rs.on('data', chunk => console.log(chunk.toString()))
rs.write('Hello W')
rs.write('orld!')
rs.end()
```
_flush方法是在流结束前被调用，该方法在流结束前会被调用。
**使用pipe拼接流**
```
readable.pipe(writable, [options])
```
不用考虑背压的问题，管道会自动处理,error事件不会在管道中自动传递

### 设计模式
(1) 工厂模式
工厂模式允许我们将对象的创建从实现中分离出来，能更好的控制对象的创建
```
var Shop = function(goods){
    if(this instanceof shop){　　　　//防止使用shop的时候将shop直接当成函数使用
        return new this[goods](); 
    }else{
        return new shop(goods);
    }
}
```
(2) 代理模式
代理(proxy)实现了一个用来控制对另一个对象访问的对象。它实现了与本体对象相同的接口，可以对两个对象进行随意替换使用。
```
function createProxy (subject) {
    const proto = Object.create(subject)
    function Proxy (subject) {
        this.subject = subject
    }
    Proxy.prototype = Object.create(proto)
    Proxy.prototype.hello = function () {
        return this.subject.hello() + 'world!'
    }
    Proxy.prototype.goobye = function () {
        return this.subject.goodbye.apply(this.subject, arguments)
    }
    return new Proxy(subject)
}
```
ES6中的Proxy对象也是一种代理模式
(3) 装饰者模式
装饰者模式是一种结构模式，用来动态为现有对象添加一些额外行为。实现方法跟代理模式一样，使用对象增强，给被装饰的对象添加一些新的方法:
```
function decorate (component) {
    component.greeting = () => {}
    return component
}
```
(4) 适配器模式
适配器模式允许我们通过一个不同的接口去使用原有对象的方法，适配器本质上是对被适配者的包装
(5) 策略模式
策略模式允许一个称为上下文的对象，将变量部分提取到独立的、可变换的策略对象中，从而支持逻辑中的变化
(6) 状态模式
策略模式的策略选择基于不同的变量，一旦选择完成，策略在上下文剩下的生命周期中不变；在状态模式中，策略师动态的，在上下文周期中是可变的，允许内部根据不同的状态来使用不同的行为。

### 连接模块
构建模块时平衡的两个最重要的特性是**内聚**和**耦合**，理想情况是高内聚和低耦合。
(1) 硬编码依赖
在Node.js中，一个客户端模块可以使用require()显示加载另一个模块，这就是一个硬编码的依赖关系。一个认证服务器的结构:
AuthController => AuthService => DB
创建一个lib/db.js文件:
```
const level = require('level')
const sublevel = require('level-sublevel')
module.exports = sublevel(
    level('example-db', {valueEncoding: 'json'})
)
```
认证服务(lib/authService.js)模块，该组件服务根据数据库中的信息检查用户凭据
```
const db = require('./db')
const users = db.sublevel('users')

const tokenSecret = 'SHHH!'
exports.login = (username, password, callback) => {
    users.get(username, function (err, user) {})
}
exports.checkToken = (token, callback) => {
    users.get(userData.username, function (err, user) {})
}
```
authConstroller模块，该模块负责处理HTTP请求:
```
const authService = require('./authService')
exports.login = (req, res, next) => {
    authService.login(req.body.username, req.body.password, (err, result) => {})
}
exports.checkToken = (req, res, next) => {
    authService.checkToken(req.query.token, (err, result) => {})
}
```
app.js入口
```
const authConstroller = require('./lib/authController')
const express = require('express')
const bodyParser = require('body-parser')
const app = express()

app.use(bodyParser.json())
app.post('/login', authController.login)
app.get('/checkToken', authController.checkToken)
...
```
硬编码依赖的优点: 这样的模块组织直观、易于理解和调试，其中每一个模块初始化和连接都无须任何外部干预;
缺点: 硬编码对有状态实例的依赖限制了模块与其他实例连接的可能，如authService与另一个数据库实例组合重用几乎是不可能的，因为它的依赖是用于一个特定的实例硬编码
(2) 依赖注入 (DI)
依赖注入模式背后的思想是通过外部实体提供作为输入的组件的依赖性 
重构lib/db.js模块，将db模块转换为工厂
```
const level = require('level')
const sublevel = require('level-sublevel')
module.exports = dbName => {
    return sublevel(
    level(dbName, {valueEncoding: 'json'})
)
}
```
重构lib/authService.js模块
```
const jwt = require('jwt-simple')
const bcrypt = require('bcrypt')
module.exports = (db, tokenSecret) => {
    const users = db.sublevel('users')
    const authService = {}
    authService.login = (username, password, callback) => {
    users.get(username, function (err, user) {})
    }
    authService.checkToken = (token, callback) => {
        users.get(userData.username, function (err, user) {})
    }
    return authService
}
```
重构lib/authController.js
```
module.exports = (authService) => {
    const authController = {}
    authController.login = (req, res, next) => {
        .body.password, (err, result) => {})
    }
    authController.checkToken = (req, res, next) => {
        authService.checkToken(req.query.token, (err, result) => {})
    }
    return authController
}
```
app.js
```
...
const dbFactory = require('./lib/db')
const authServiceFactory = require('./lib/authService')
const authControllerFactory = require('./lib/authController')
const db = dbFactory('example-db')
const authService = authServiceFactory(db, 'SHHH!')
const authController = authControllerFactory(authService)
app.post('/login', authController.login)
app.get('/checkToken', authController.checkToken)
```
DI的优点: 可以轻易地重用每一个模块，而且测试一个使用DI模式的模块工作也大大简化
缺点: 在编码时不能解决依赖问题，使得我们难以理解系统的各个组件之间的关系

(3) 服务定位器
它的核心原则是，有一个中央注册表，用于管理系统的组件，并在每个模块需要加载依赖时充当调节器。
服务定位器的模块，lib/serviceLocator.js
```
module.exports = function () {
    const dependencies = {}
    const factories = {}
    const serviceLocator = {}
    serviceLocator.factory = (name, factory) => {
        factories[name] = factory
    }
    serviceLocator.register = (name, instance) => {
        dependencies[name] = instance
    }
    serviceLocator.get = (name) => {
        if (!dependencies[name]) {
            const factory = factories[name]
            dependencies[name] = factory && factory(serviceLocator)
            if (!dependencies[name]) {
                throw new Error('Cannot find module: ' + name)
            }
        }
        return dependencies[name]
    }
    return serviceLocator
}
```
lib/db.js
```
const level = require('level')
const sublevel = require('level-sublevel')
module.exports = (serviceLocator) => {
    const dbName = serviceLocator.get('dbName')
    return sublevel(
    level(dbName, {valueEncoding: 'json'})
)
}
```
lib/authService.js
```
module.exports = (serviceLocator) => {
    const db = serviceLocator.get('db')
    const tokenSecret = serviceLocator.get('tokenSecret')
    const users = db.sublevel('users')
    const authService = {}
    authService.login = (username, password, callback) => {
    users.get(username, function (err, user) {})
    }
    authService.checkToken = (token, callback) => {
        users.get(userData.username, function (err, user) {})
    }
    return authService
```
lib/authController.js
```
module.exports = (serviceLocator) => {
    const authService = serviceLocator.get('authService')
    const authController = {}
    authController.login = (req, res, next) => {
        .body.password, (err, result) => {})
    }
    authController.checkToken = (req, res, next) => {
        authService.checkToken(req.query.token, (err, result) => {})
    }
    return authController
}
```
app.js
```
const svcLoc = require('./lib/serviceLocator')()
svcLoc.register('dbName', 'example-db')
svcLoc.register('tokenSecret', 'SHHH!')
svcLoc.factory('db', require('./lib/db'))
svcLoc.factory('authService', require('./lib/authService'))
svcloc.factory('authController', require('./lib/authController'))

const authController = svcLoc.get('authController')

app.post('/login', authController.login)
app.all('/checkToken', authController.checkToken)
```
定位服务器的优点: 将依赖项所有权转移到组件的外部实体
缺点: 很难识别组件之间的关系，需要使用代码来检查或在文档中使用明显的语句来解释一个特定的组件会加载什么依赖

(4) 依赖注入容器
DI容器本质上是一个服务定位器，其增加一个特性: 在实例化之前标识模块的依赖需求，用函数参数来获取依赖名称。
在lib/directory目录下创建一个名为dicontainer.js的新模块
```
const fnArgs = require('parse-fn-args')
module.exports = function () {
    const dependencies = {}
    const factories = {}
    const diContainer = {}
    diContainer.factory = (name, factory) => {
        factories[name] = factory
    }
    diContainer.register = (name, instance) => {
        dependencies[name] = instance
    }
    diContainer.get = (name) => {
        if (!dependencies[name]) {
            const factory = factories[name]
            dependencies[name] = factory && diContainer.inject(factory)
            if (!dependencies[name]) {
                throw new Error('Cannot find module: ' + name)
            }
        }
        return dependencies[name]
    }
    diContainer.inject = (factory) => {
        const args = fnArgs(factory)
        .map(dependency => diContainer.get(dependency))
        return factory.apply(null, args)
    }
    return diContainer
}
```
DI容器的优点: 更好的解耦性和可测试性，它不强制模块依赖任何额外的服务，除了它的实际依赖
缺点: 具有更高的复杂性，因为它依赖在运行时解析