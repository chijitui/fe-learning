### 引言
这是一个老生常谈的经典面试问题了，它从`ES5`的时代就开始被关注，直到现在都还会被提及。不管你用什么框架，用什么开发方式，你基本都不太可能直接绕过`this`这个东西。

有些时候可能多用或者少用一个`箭头函数`，代码的输出就完全乱了。既然提到了面试，那就从一个面试题开始吧：

```javascript
var fullname = 'john Doe';
var obj = {
    fullname: 'colin Mork',
    prop: {
        fullname:'rosa Kus',
        getFullname: function() {
            return this.fullname;
        }
    }
}

console.log(obj.prop.getFullname()); // 结果________
var getfullname = obj.prop.getFullname;
console.log(getfullname()); // 结果________
```
可以先思考一下这东西输出什么。

### this 的指向
其实关于`this`的指向，刚学 JS 的时候老被一些博客坑，它们总会说很多种情况，最后总结得我都看不懂说什么，只能背下每种情况。

但实际上并没有那么复杂，只需要坚持一个原理就行了：**`this`指向最后调用它的那个对象**。没错，就是这么简单。

来看下前面的这道题：
- 由于`console.log(obj.prop.getFullname())`调用`getFullname()`的对象是`prop`，所以第一个`this`指向`prop`，输出的就是`'rosa Kus'`；
- 然后`console.log(getfullname())`中是一个全局的调用，因此`this`指向全局，在浏览器中指向`window`，输出`'john Doe'`，而在 Node.js 环境下指向`global`，输出的是`undefined`。

**题外话：** 代码中的`var fullname = 'john Doe'`在浏览器中可以直接挂载到全局变量`window`下，但是在 Node.js 中全局变量`global`是不接受这样的挂载的，只能通过`global.fullname = 'john Doe'`挂载上去，这也就是为什么上面会输出`undefined`。

**注意：匿名函数的`this`通常指向全局，因为它是挂载在全局上的。**

## 箭头函数
> **箭头函数** 的`this`始终指向函数定义时的`this`，而非执行时。

关于箭头函数，只需要了解它不会生成新的`this`指向，它会寻找函数调用时的作用域里`this`指向，在用箭头函数改造原有函数的时候，一定要注意。比如上面的题目改造一下，它的`this`是指向全局的：
```javascript
var fullname = 'john Doe';
var obj = {
    fullname: 'colin Mork',
    prop: {
        fullname:'rosa Kus',
        getFullname: () => {
            return this.fullname;
        }
    }
}

// 浏览器：=> 'john Doe'
console.log(obj.prop.getFullname());
```

### 关于 call / apply / bind

虽然是`ES5`里面的东西，但这三个方法到现在也不过时，依旧有很多的代买是通过他们去修改`this`的指向。

那我们就依次聊一下，并用`ES6`的方法去实现一下：
#### 1. call
关于`call`的使用，先举个例子：
```javascript
function add(...arr) {
    return arr.reduce((a, b) => {
        return a + b 
    }, this.c);
}
const newObj = { c: 1 };
add.call(newObj, 2, 3); // 6
```
好的来解释一下`call`做了什么，实际上它就是将`add`函数中的`this`指向了`newObj`并且执行了`add`函数。所以就来自己实现一下这个方法：

```javascript
Function.prototype.call = function(context, ...args) {
    const ctx = context || window;
    ctx.func = this;
    const result = ctx.func(...args);
    delete ctx.func;
    return result;
};
```

#### 2. apply
`apply`和`call`做的事情是差不多的，只不过传入的参数不同：
```javascript
add.apply(newObj, [2, 3, 6]); // 12
```
那么它的实现也就很简单了，比`call`少一个参数合并的步骤就行了：
```javascript
Function.prototype.apply = function(context, args) {
    const ctx = context || window;
    ctx.func = this;
    const result = ctx.func(...args);
    delete ctx.func;
    return result;
};
```

#### 3. bind
一样先看例子：
```javascript
function foo(c, d) {  
    this.b = 100;
    console.log(c);
    console.log(d);
    console.log(this.a);
    console.log(this.b);
}  
const func = foo.bind({ a: 1 }, 'cc', 'dd');
func(); // cc dd 1 100
new func(); // cc dd undefined 100 并且实例化一个 { b: 100 } 的对象
```
对于`bind`函数，它会生成一个新函数，第一个参数作为运行时的`this`，其它参数会作为参数传入函数中；但是在作为构造函数的时候，`bind`所指定的`this`会失效。实现如下：
```javascript
Function.prototype.bind = function(context, ...rest) {
    if (typeof this !== 'function') {
        throw new TypeError('invalid invoked!');
    }
    const self = this;
    return function F(...args) {
        if (this instanceof F) {
            return new self(...rest, ...args);
        }
        return self.apply(context, rest.concat(args));
    }
};
```

### 构造函数中的 this
我们经常用关键字`new`来实例化一个对象，关于构造函数中的`this`，我觉得聊清楚了`new`一个对象的过程就能懂了，虽然`new`不出一个小姐姐，但是可以`new`出一个`this`：
- 创建一个空对象；
- 将构造函数的作用域赋给新对象，`this`的指向新对象；
- 执行构造函数中的代码，为这个新对象添加属性；
- 返回新对象。

所以啊，`this`就自然指向实例化出的对象了。