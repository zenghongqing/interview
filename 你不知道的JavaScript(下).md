### 块作用域
(1) let 声明
* let声明属于块级作用域,  但是直到在块中出现才会被初始化
* 过早访问let声明的引用导致的这个ReferenceError严格说叫做临时死亡区(TDZ), 应该把所有的let声明放在其所在作用域最前面
* for循环头部的let i不只为for循环本身声明了一个i,而是为循环的每一次迭代都重新声明一个新的i
(2) const声明
* const 声明必须要有显示的初始化
* 将对象或者数组作为常量赋值, 意味着这个值在这个常量的词法作用域结束前不会被垃圾回收, 因为指向这个值的引用没有清除
* 常量不是对这个值本身的限制, 而是对赋值的那个变量的限制

### 默认参数值
undefined意味着缺失, undefined和缺失是无法区别的, 至少对于函数参数来说是如此。
```
function foo (x = 11, y = 31) {
    console.log(x + y)
}
foo(5, undefined) // 36 <-- 丢了undefined
foo(5, null) // 5 <-- null被强制转化为0
```
##### 默认值表达式
默认值表达式是惰性求值的, 意味着只在需要的时候运行--是在参数的值省略或者为undefined时
```
var w = 1, z = 2
function foo (x = w + 1, y = x + 1, z = z + 1) {
    console.log(x, y, z)
}
foo() // ReferenceError
```
注意函数声明中形式参数是在它们自己的作用域中, 而不是在函数体作用域中。意味着默认值参数表达式中的标识符引用首先匹配到形式参数作用域, 然后才会搜索外层作用域。

### 解构
* 对于对象结构形式来说, 如果省略了var/let/const声明符, 就必须把整个表达式用()括起来。如果不这么做, 语句左侧的{..}会被作为语句中的第一个元素就会被当做是一个块语句而不是一个对象
```
var a, b, c, x, y, z
[a, b, c] = foo()
({x, y, z} = bar())
```
不用临时变量解决‘交换两个变量’
```
var x = 10, y = 20
[y, x] = [x, y]
```
* 对象解构形式允许多次列出同一个源属性(持有值类型任意)
默认值赋值
```
var x = 200, y = 300, z = 100
var o1 = {x: {y: 42}, z: {y: z}}
({y: x = {y: y}} = o1) // 因为o1中无y属性, x.y = y = 300
({z: y = {y: z}} = o1) // 因为o1中z属性, y.y = z = 100
({x: z = {y: x}} = o1) // 因为o1中x属性, z.y = 42
```
* 解构默认值+参数默认值
```
function f6({x = 10} = {}, {y} = {y: 10}) {
    console.log(x, y)
}
f6() // 10, 10
f6({}, {}) // 10. undefined
f6(undefined, undefined) // 10, 10
f6({}, undefined) // 10, 10
f6(undefined, {}) // 10, undefined
```
### 对象字面量扩展
* 计算属性名
```
var prefix = 'user_'
var o = {
    [prefix + 'foo']: function () {},
    [Symbol.toStringTag]: 'really cool thing'
}
```
* 设定[[prototype]]
使用ES6工具Object.setPrototypeOf()
* super对象
```
var o1 = {
    foo () {
        console.log('o1: foo)
    }
}
var o2 = {
    foo () {
        super.foo()
        console.log('o2:foo')
    }
}
Object.setPrototypeOf(o2, o1)
o2.foo() // o1: foo
         // o2: foo 
```

### 箭头函数
* 箭头函数总是函数表达式, 箭头函数是匿名函数表达式--它们没有用于递归或者事件绑定/解绑定的命名引用
* 箭头函数的this, arguments继承自父层

### 符号
* 不能也不应该对Symbol使用new。它并不是一个构造器, 也不会创建一个对象
* 传给Symbol(..)的参数是可选的.。如果传入了的话, 应该是一个为这个symbol的用途给出友好描述的字符串
* symbol不是Symbol的实例, 出于某种原因想要构造一个symbol值的装箱封装对象形式
```
var sym = Symbol('some optional desc')
sym instanceof Symbol // false
var symObj = Object(sym)
symObj instanceof Symbol // true
symObj.valueOf() === sym // true
```
* 符号的主要意义是创建一个类字符串的不会与其他任何值冲突的值
* 符号注册
```
var EVENT_LOGIN = Symbol.for('event.login')
console.log(EVENT_LOGIN)
```
Symbol.for(..)在全局符号注册表中搜索,来查看是否有描述文字相同的符号已经存在, 存在则返回它。
* 作为对象属性的符号
```
var o = {
    foo: 42,
    [Symbol('bar')]: 'hello world',
    baz: true
}
// 要取得对象的符号属性
Object.getOwnPropertySymbols(o)
```

### 生成器
生成器在其中使用新的关键字yield(优先级很低), 暂停自身
```
function *foo() {
    while (true) {
        yield Math.random()
    }
}
```
因为yield *可以调用另一个生成器(通过委托到其迭代器)
```
function *foo (x) {
    if (x < 3) x = yield *foo(x + 1)
    return x * 2
}
foo(1) // 24
```

### ES6模块
* ES6使用基于文件的模块, 也就是说一个文件一个模块
* ES6模块的API是静态的
* ES6模块是单例, 如果要产生多个模块实例, 则你的模块需要提供某种工厂函数来实现
* 模块的公开API中暴露的属性和方法并不仅仅是普通的值或是引用的赋值。它们是到内部模块定义中的标识符的实际绑定
* 导入模块与静态请求这个模块是一样的

### 类
```
class Foo {
    constructor (a, b) {
        this.x = a
        this.y = b
    }
    gimmeXY () {
        return this.x * this.y
    }
}
// entends继承
class Bar extends Foo {
    constructor (a, b, c) {
        super(a, b)
        this.z = c
    }
    gimmeXYZ () {
        return super.gimmeXY() * this.z
    }
}
var b = new Bar(5, 15, 25)
```
* class 调用必须通过new来实现
* class Foo不能变量提升
* Bar extends Foo意思是把Bar.prototype的[[prototype]]连接到Foo.prototype, super指向Foo.prototype, 而在Bar构造器中super指的是Foo
* super是静态绑定到类层次的
* 子类构造器中super()调用后才能访问this
* 在任何构造器中, new.target总是指向new 实际上调用的构造器, 即使构造器是在父类中且通过子类构造器用super(..)委托调用
* static方法是直接添加到这个类的函数对象上, 而不是这个函数对象的prototype对象上
* Symbol.species Getter构造器
```
class Foo {
    // 推迟species为子构造器
    static get [Symbol.species] () {return this}
    spawn () {
        return new constructor[Symbol.species]()
    }
}
class Bar extends Foo {
    static get [Symbol.species] () {return Array}
}
var a = new Foo ()
var b = a.spawn()
b instanceof Foo // true

var x = new Bar()
var y = x.spawn()
y instanceof Bar // false
y instanceof Foo // true
```

### Promise
Promise是一种封装和组合未来值的易于复制的机制
Promise的局限性
* 顺序错误处理: Promise链中的错误很容易被无意中默默忽略掉
* 单一值: Promise只能有一个完成值或一个拒绝理由, 一般建议构造一个值封装来保持这样的多个信息
* 单决议: Promise只能被决议一次
* 无法取消的Promise: 一旦创建了一个Promise并为其注册了完成或拒绝函数, 如果出现某种情况使得这个任务悬而未决, 你也没办法从外部停止它的进程
* Promise的性能: 因为Promise是所有的一切都成为了异步, 即有一些立即(同步)完成的步骤仍然会延迟到任务的下一步, 意味着会稍慢一些,  但是能得到大量内建的可信任性、对Zalgo的避免以及可组合性。