许多面向对象的语言都有 **装饰器(Decorator)** 函数，用来修改类的行为。目前，这个方法已经被引入了 **ES7**，但是无论是主流浏览器还是 Node.js 对它的兼容性都不是特别友好。

因此要在项目中使用`Decorator`的话，需要使用 **Babel** 进行转移，或者使用 Javascript 的超集 **Typescript** 来进行开发。

> 如果对这一语法细节还不是很了解的话，可以先进这个传送门：http://es6.ruanyifeng.com/#docs/decorator ，跟着阮一峰老师一起了解一下它的特性。

### 初衷
使用 **装饰器** 的初衷来自于不想修改原来接口的情况下，能让一件事表现得更好。就像：
- 手机可以用，但是加了手机壳就能防摔；
- 椅子可以坐，但是垫了垫子就能够坐的更舒服；
- 步枪可以射击，但是加了瞄准镜就可以射的更准；
- ......

如果要更加抽象地理解，在计算机领域，它就可以被应用到日志收集、错误捕获、安全检查、缓存、调试、持久化等等方面。

### 常用的装饰器
常用的装饰器一般有 **类装饰器** 和 **方法装饰器**，当然也会有**属性装饰器**，但是用的不多就不多讨论了。
#### 类装饰器
主要应用于类构造函数，其参数是类的构造函数：
```javascript
function testable(target) {
    target.prototype.isTestable = true
}

@testable
class MyTestableClass {}

let obj = new MyTestableClass()
obj.isTestable // true
```
**注意：** 这里的`target`参数如果直接给它添加方法，获得的是一个静态方法，相当于在`class`的方法前添加`static`关键字；如果想添加实例属性，可以通过目标类的`prototype`对象操作。

现在我们就用 **类装饰器** 实现一个捕获方法执行时间的装饰器：
```javascript
const sleepTimeClass = (timeHandler?: (time?: number) => void) => (target: any) => {
    Object.getOwnPropertyNames(target.prototype).forEach(key => {
        const func = target.prototype[key]
        target.prototype[key] = async (...args: any[]) => {
            const startTime = await +new Date()
            await func.apply(this, args)
            const endTime = await +new Date()
            timeHandler && await timeHandler(endTime - startTime)
        }
    })
    return target
}
```
之所以还在外面包了一层函数，是为了通过柯里化，让使用者可以再进一步处理得到的执行时间：
```javascript
const sleepTimeClassTimer = sleepTimeClass(time => {
    console.log('执行时间', `${time}ms`)
})

@sleepTimeClassTimer
class homepageController {
    async get(ctx: any) {
        ctx.response.body = await pageService.homeHtml('/page/helloworld', '/page/404')
    }
}
```

这样，每次`class`中的方法执行完之后就会打印出相应的执行时间。

#### 方法装饰器
```javascript
function readonly(target, name, descriptor){
    // descriptor对象原来的值如下
    // {
    //   value: specifiedFunction,
    //   enumerable: false,
    //   configurable: true,
    //   writable: true
    // }
    descriptor.writable = false
    return descriptor
}

readonly(Person.prototype, 'name', descriptor)
// 类似于
Object.defineProperty(Person.prototype, 'name', descriptor)
```
由于在异步编程的时候，`async`和`await`的异常很难捕获，如果强行用`try...catch`来搞，捕捉不完不说，代码看起来还很难看，使用装饰器就很简单了：
```javascript
const asyncMethod = (errorHandler?: (error?: Error) => void) => (...args: any[]) => {
    const func = args[2].value
    return {
        get() {
            return (...args: any[]) => {
                return Promise.resolve(func.apply(this, args)).catch(error => {
                    errorHandler && errorHandler(error)
                })
            }
        },
        set(newValue: any) {
            return newValue
        }
    }
}
```

接着使用方法装饰器：
```javascript
const errorAsyncMethod = asyncMethod(error => {
    console.error('错误警告', error)
})

class homepageController {
    @errorAsyncMethod async get(ctx: any) {
        ctx.response.body = await pageService.homeHtml('/page/helloworld', '/page/404')
    }
}
```

### 装饰器加载顺序
一个类或者方法可以嵌套很多个装饰器，所以搞清楚它们的执行顺序也很重要：
- 有多个参数装饰器时，从最后一个参数依次向前执行；
- 方法和方法参数中参数装饰器先执行；
- 类装饰器总是最后执行；
- 方法和属性装饰器，谁在前面谁先执行；
- 因为参数属于方法一部分，所以参数会一直紧紧挨着方法执行。

### 装饰器的应用
在初衷那里就已经提到了，试着想象一下，只需要几个装饰器就可以完成前后端基本的性能和日志监控，是不是很有意思？