很久之前在`知乎`上看到一个问题：
> 为什么 Node.js 不给每一个.js 文件以独立的上下文来避免作用域被污染?

后来在`饿了么`的分享中也碰到了相应的问题，就来整理一下，希望回顾的同时也能帮到需要的同学，~~如果这个没弄懂确实不应该说自己熟悉 Node.js 了~~。

关于这个问题，首先可以想到的关键词就是`模块机制`了，所以就先来整理一下 Node.js 的模块机制，最重要的就是`require`的原理。先看一下`require`的实现原理：
```javascript
function require(...) {
    var module = { exports: {} };
    ((module, exports) => {
        // Your module code here. In this example, define a function.
        function some_func() {};
        exports = some_func;
        // At this point, exports is no longer a shortcut to module.exports, and
        // this module will still export an empty default object.
        module.exports = some_func;
        // At this point, the module will now export some_func, instead of the
        // default object.
    })(module, module.exports);
    return module.exports;
}
```
由于这样的机制，我们再关注一下 Node.js 对 CommonJS 的实现：
```javascript
var wrapper = Module.wrap(content);

var compiledWrapper = vm.runInThisContext(wrapper, {
    filename: filename,
    lineOffset: 0,
    displayErrors: true
});

// ...

var result = compiledWrapper.call(this.exports, this.exports, require, this, filename, dirname);
```

在源码中，`content`实际上就是模块中的代码，比如你传如一个`'console.log(t);'`，在`Module.wrap`完之后就会得到一个字符串`wrapper`：
```javascript
'(function (exports, require, module, __filename, __dirname) { console.log(t);\n});'
```
然后用`vm.runInThisContext`将字符串转化成：
```javascript
(function(exports, require, module, __filename, __dirname) {
    console.log(t);
})();
```
在`vm.runInThisContext`的官方文档中对于编译之后的作用域解释为：
> 在指定的`global`对象的上下文中执行`vm.Script`对象里被编译的代码并返回其结果。被执行的代码虽然无法获取本地作用域，但是能获取`global`对象。

这解释了以下两个情况：
- `vm.runInThisContext`使得包裹函数执行时无法影响本地作用域；
- `global`对象是可以访问的。

也解释了`饿了么`提出的问题：
> 如果 a.js `require`了 b.js, 那么在 b 中定义全局变量`t = 111`能否在 a 中直接打印出来？

答案是可以的，因为`global`对象是可以访问的，因此`t = 1`等价于 `global.t = 111`，但是如果声明了`t`，上下文就不会被污染。

回到最初的问题，由于只有在 .js 文件中没有声明过才会被挂载在`global`对象上，每个模块就是独立的作用域，所以要避免上下文污染，只需要在写代码的时候添加`'use strict'`：
```javascript
'use strict'
t = 111 // 报错 t 未定义。
```
最后有人问就可以总结成一句话：
> Node.js 模块正常情况对作用域不会造成污染，意外创建全局变量是一种例外，可以采用严格模式来避免。
