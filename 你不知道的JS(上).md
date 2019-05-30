### 作用域
* 引擎
从头到尾负责整个JavaScript程序的编译和执行
* 编译器
负责语法分析和代码生成:
(1) 分词/词法分析
(2) 解析/语法分析
(3) 代码生成
* 作用域
负责收集并维护由所有声明的标识符(变量)组成的一系列查询, 并实施一套严格的规则, 确定当前执行的代码对这些标识符的访问权限
#### 理解作用域
变量的赋值操作会执行两个动作, 首先编译器会在当前作用域声明一个变量(如果之前没声明过), 然后在运行时引擎会在作用域中查找该变量, 如果能找到就会对它赋值，如果未找到则会沿着作用域链继续查找, 直到找到。如果未找到引擎会抛出一个异常。
####  作用域嵌套
当一个块或函数嵌套在另一个块或函数中时, 就发生了作用域嵌套。遍历嵌套作用域链的规则: 引擎从当前的执行作用域开始查找变量, 如果找不到, 就向上一级继续查找。当抵达最外层的全局作用域时, 无论找到还是没找到, 查找过程都会终止。

### 词法作用域
* 定义在词法阶段的作用域; 无论函数在哪里被调用, 也无论如何被调用, 它的词法作用域都只由函数被声明时所处的位置决定。
* 词法作用域查找只会查找一级标识符, 比如a、b、c。如果代码中引用了foo.bar.baz, 词法作用域查找只会试图查找foo标识符, 找到这个变量后, 对象属性访问规则会分别对bar和baz属性访问
* 欺骗词法作用域会导致性能下降: eval和with, 这两个机制的副作用是引擎无法在编译时对作用域查找进行优化, 因为引擎只能谨慎的认为这样的优化是无效的。使用其中任何一个机制都会导致代码运行缓慢。

### 函数作用域和块作用域
函数作用域: 属于这个函数的全部变量都可以在整个函数的范围内使用和复用(事实上在嵌套的作用域内也可以使用)

### 提升
函数声明和变量声明都会被提升, 且函数会首先被提升, 然后才是变量(如果变量名与函数名相同则被忽略)

### 作用域闭包
闭包: 一个函数中定义了另一个内部函数, 无论通过何种手段将内部函数传递到所在的词法作用域以外, 它都会对原始定义作用域的引用, 无论在何处执行这个函数都会使用闭包。

### this指向
要判断一个运行函数的this指向, 需要找到这个函数的直接调用位置。找到后按照如下顺序来判断this指向:
* 由new调用? 绑定到新建的对象
* 由call或者apply(或者bind)调用? 绑定到指定的对象
* 由上下文对象调用? 绑定到那个上下文对象
* 默认: 在严格模式下绑定到undefined, 否则绑定到全局对象
call、apply或者bind总是使用null来忽略this绑定可能会产生一些副作用: 如果某个函数确实使用了this(如第三方库中的一个函数), 那默认规则会把this绑定到全局对象, 这将导致不可预计的后果, 可以使用一个DMZ对象,如o = Object.create(null), 以保护全局对象
ES6中的箭头函数是根据当前的词法作用域来决定this, 具体来说会继承外层函数调用的this绑定。

### 对象
1. 语法: 对象可以通过两种形式定义: 声明形式和构造形式
```
var myObj = {key: value}
和 var myObj = new Object()
```
2. 基本类型: string、number、boolean、null、undefined、object、symbol
3. 内容: 存储在对象容器内部的是属性的名称, 像指针(引用)一样, 指向这些值真正的存储位置; 通过.propName或者['propName']语法来获取属性值, 访问属性时, 引擎实际上会调用内部的默认[[Get]]操作(设置属性时是[[Put]]), [[Get]]操作会检查对象本身是否包含这个属性, 如果没找到的话还会查找[[Prototype]]链
4. 属性描述符
* writable 决定是否可以修改属性的值
```
var myObject = {}
Object.defineProperty(myObject, 'a', {
    value: 2,
    writable: false,
    configurable: true,
    enumerable: true
})
myObject.a = 3
myObject.a; // 2
```
* configurable 只要属性是可配置的, 就可以使用defineProperty方法来修改属性描述符, 把configurable修改成false是单向操作, 无法撤销; configurable: false还会禁止(delete)删除这个属性
* enumable 设置成false, 这个属性不会出现在枚举如for..in中, 虽然仍然可以正常访问它。
使用ES6的for..of来遍历数据结构(数组、对象等)中的值, 它会寻找内置或自定义的@@iterator对象并调用它的next()方法来遍历数据值。

### 类和原型
如果要访问对象中不存在的属性, [[Get]]操作就会查找对象内部[[Prototype]]关联的对象。这个关联关系实际上定义了一条原型链, 在查找属性时会遍历它。
(1) 如果foo不直接存在于myObject中, 而是存在于原型链上层时myObject.foo = "bar"会出现三种情况:
* 如果在[[Prototype]]链上层存在名为foo的普通数据访问属性, 并且没被标记只读(writable: false), 那就会直接在myObject中添加一个名为foo的新属性, 它是屏蔽属性。
* 如果在[[Prototype]]链上层存在foo, 但是它被标记为只读(writable: false), 那么无法修改已有属性或者在myObject上创建属性屏蔽。如果在严格模式下, 代码会抛出一个错误。否则这条赋值语句会被忽略。总之不能屏蔽。
* 如果在[[Prototype]]链上层存在foo并且设置一个setter, 那就一定会调用这个setter。foo不会被添加到myObject, 也不会重新定义foo这个setter。
(2) Object.create的polyfill代码
```
if (!Object.create){
    Object.create = function (o) {
        function F () {}
        F.prototype = o
        return new F()
    }
}
```
(3) 关联关系
```
var anotherObject = {
    cool: function () {
        console.log('cool!)
    }
}
var myObject = Object.create(anotherObject)
myObject.doCool = function () {
    this.cool() // 内部委托
}
myObject.doCool()
```
### ES6中的class
class的优点
* 不再引用杂乱的.prototype
* 不再需要Object.create(..)来替换.prototype对象, 也不需要设置.__proto__或者Object.setPrototypeOf(..)
* 可以通过super(..)来实现相对多态, 这样任何方法都可以引用原型链上层的同名方法; 构造函数不属于类, 所以无法相互引用---super()可以完美解决构造函数的问题
* class字面语法不能声明属性(只能声明方法)
* 可以通过extends很自然地扩展对象子类型, 甚至是内置的对象类型, 如Array或RegExp

class 缺陷
* class无法定义类成员属性
* 每次执行操作都必须重新绑定super, super属于静态绑定

### 常见的四种内存泄露
* 以外的全局变量
```
function foo (arg) {
    bar = 'this is a hidden global variable'
}
foo()
```
函数调用完成后，函数内部的局部变量会被垃圾回收机制清理, 但是内部定义的是全局变量，所以该内存会一直被占用

* 被遗忘的计时器或者回调函数
```
var someResource = getData()
setInterval(function(){
    var node = document.getElementById('Node')
    if (node) {
        // 处理 node 和 someResource
        node.innerHTML = JSON.stringify(someResource))
    }
}, 1000)
```
该例子表明node或者数据不再需要时，定时器依旧指向这些数据; 就算把node节点清除，setInterval依旧存活且垃圾回收器无法回收, 唯一的办法就是停止定时器
```
var element = document.getElementById('button')
function onClick () {
    element.innerHTML = 'text;
}
element.addEventListener('click', onClick)
```
老的IE6是无法处理循环引用的, 因为老版本的 IE 是无法检测 DOM 节点与 JavaScript 代码之间的循环引用，会导致内存泄漏;但是现代浏览器采用了垃圾回收算法可以正确检测和处理循环引用了。即回收节点内存时, 不必非要调用removeEventListener了。

* 脱离DOM的引用
```var elements = {
    button: document.getElementById('button'),
    image: document.getElementById('image'),
    text: document.getElementById('text')
}
function doStuff () {
    image.src = 'http://some.url/image'
    button.click()
    console.log(text.innerHTML)
}
function removeButton() {
    // 按钮是 body 的后代元素
    document.body.removeChild(document.getElementById('button'));
    // 此时，仍旧存在一个全局的 #button 的引用
    // elements 字典。button 元素仍旧在内存中，不能被 GC 回收。
}
```
同样的DOM元素存在两个引用: 一个在DOM树中,另一个在字典中。将来需要把两个引用都清除。如果代码中保存了表格某一个<td>的引用。将来决定决定删除整个表格的时候,GC并不会回收已保存的<td>以外的其他节点,此<td>是表格的子节点,子元素与父元素是引用关系。由于代码保留了<td>的引用,导致整个表格仍在内存中。
* 闭包
```
var theThing = null
var replaceThing = function () {
    var originThing = theThing
    var unused = function () {
        if (originThing) {
            console.log('Hi')
        }
    }
    theThing = {
        longStr: new Array(1000000).join('*),
        someMethod: function () {
            console.log(someMessage)
        }
    }
}
setInteval(replaceThing, 1000)
```
每次调用replaceThing, theThing得到一个包含一个大数组和一个新闭包(someMethod)的新对象。同时，变量 unused 是一个引用 originalThing 的闭包（先前的 replaceThing 又调用了 theThing ）。someMethod可以通过theThing使用,someMethod与unused分享闭包作用域, 尽管unused从未使用,它引用的originalThing迫使它保留在内存中防止被回收。
解决办法: 在 replaceThing 的最后添加 originalThing = null