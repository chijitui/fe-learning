### Node.js 风格函数的 promise 化
在 Javascript 中， 并非所有的异步控制函数和库都支持开箱即用的`promise`,所以在大多数情况下都需要吧一个典型的基于回调的函数转换成一个返回`promise`的函数，比如：
```javascript
function promisify(callbackBaseApi) {
    return function promisified() {
        const args = [].slice.call(arguments);
        return new Promise((resolve, reject) => {
            args.push((err, result) => {
                if(err) {
                    return reject(err);
                }
                if(arguments.length <= 2) {
                    resolve(result);
                } else {
                    resolve([].slice.call(arguments, 1));
                }
            }); 
            callbackBaseApi.apply(null, args);
        });
    }
}
```
现在的 Node.js 核心实用工具库`util`里面已经支持`(err, value) => ...`回调函数是最后一个参数的函数, 返回一个返回值是一个`promise`版本的函数。

### 顺序执行流的迭代模式
在看异步控制流模式之前，先开始分析顺序执行流。按顺序执行一组任务意味着一次运行一个任务，一个接一个地运行。执行顺序很重要，必须保留，因为列表中任务运行的结果可能影响下一个任务的执行，比如：
```
start -> task1 -> task2 -> task3 -> end
```
这种流程一般都有着几个特点：
- 按顺序执行一组已知任务，而没有链接或者传播结果；
- 一个任务的输出作为下一个的输入；
- 在每个元素上运行异步任务时迭代一个集合，一个接一个。

这种执行流直接用在阻塞的 API 中并没有太多问题，但是，在我们使用非阻塞 API 编程的时候就很容易引起回调地狱。比如：

```javascript
task1(err, (callback) => {
    task2(err, (callbakck) => {
        task3(err, (callback) => {
            callback();
        });
    });
});
```
传统的解决方案是进行任务的拆解方法就是把每个任务拆开，通过抽象出一个迭代器，在任务队列中去顺序执行任务：
```javascript
class TaskIterator {
    constructor(tasks, callback) {
        this.tasks = tasks;
        this.index = 0;
        this.callback = callback;
    }
    do() {
        if(this.index === this.tasks.length) {
            return this.finish();
        }
        const task = tasks[index];
        task(() => {
            this.index++;
            this.do();
        })
    }
    finish() {
        this.callback();
    }
}

const tasks = [task1, task2, task3];
const taskIterator = new TaskIterator(tasks, callback);

taskIterator.do();
```

需要注意的是, 如果`task()`是一个同步操作的话，这样执行任务就可能变成一个递归算法，可能会有由于不再每一个循环中释放堆栈而达到调用栈最大限制的风险。

> **顺序迭代** 模式是非常强大的，因为它可以适应好几种情况。例如，可以映射数组的值，可以将操作的结果传递给迭代中的下一个，以实现 reduce 算法，如果满足特定条件，可以提前退出循环，甚至可以迭代无线数量的元素。 ------ Node.js 设计模式（第二版）

值得注意的是，在 ES6 里，引入了`promise`之后也可以更简便地抽象出顺序迭代的模式：
```javascript
const tasks = [task1, task2, task3];

const didTask = tasks.reduce((prev, task) =>{
    return prev.then(() => {
        return task();
    })
}, Promise.resolve());

didTask.then(() => {
    // do callback
});
```

### 并行执行流的迭代模式
在一些情况下，一组异步任务的执行顺序并不重要，我们需要的仅仅是任务完成的时候得到通知，就可以用并行执行流程来处理，例如：
```
      -> task1
     /
start -> task2     (allEnd callback)
     \
      -> task3
```
不作要求的时候，在 Node.js 环境下编程就可开始放飞自我：
```javascript
class AsyncTaskIterator {
    constructor(tasks, callback) {
        this.tasks = tasks;
        this.callback = callback;
        this.done = 0;
    }
    do() {
        this.tasks.forEach(task => {
            task(() => {
                if(++this.done === this.tasks.length) {
                    this.finish();
                }
            });
        });
    }
    finish() {
        this.callback();
    }
}

const tasks = [task1, task2, task3];
const asyncTaskIterator = new AsyncTaskIterator(tasks, callback);

asyncTaskIterator.do();
```

使用`promise`也就可以通过`Promise.all()`来接受所有的任务并且执行：
```javascript
const tasks = [task1, task2, task3];

const didTask = tasks.map(task => task());

Promise.all(didTask).then(() => {
    callback();
})
```

### 限制并行执行流的迭代模式
并行编程放飞自我自然是很爽，但是在很多情况下我们需要对并行队列的数量做限制，从而减少资源消耗，比如我们限制并行队列最大数为`2`：
```
      -> task1 -> task2 
     /
start                      (allEnd callback)
     \
      -> task3
```

这时候，就需要抽象出一个并行队列，在使用的时候对其实例化：
```javascript
class TaskQueque {
    constructor(max) {
        this.max = max;
        this.running = 0;
        this.queue = [];
    }
    push(task) {
        this.queue.push(task);
        this.next();
    }
    next() {
        while(this.running < this.max && this.queue.length) {
            const task = this.queue.shift();
            task(() => {
                this.running--;
                this.next();
            });
            this.running++;
        }
    }
}

const tasks = [task1, task2, task3];
const taskQueue = new TaskQueue(2);

let done = 0, hasErrors = false;
tasks.forEach(task => { 
    taskQueue.push(() => {
        task((err) => {
            if(err) {
                hasErrors = true;
                return callback(err);
            }
            if(++done === tasks.length && !hasError) {
                callback();
            }
        });
    });
});
```

而用`promise`的处理方式也与之相似：
```javascript
class TaskQueque {
    constructor(max) {
        this.max = max;
        this.running = 0;
        this.queue = [];
    }
    push(task) {
        this.queue.push(task);
        this.next();
    }
    next() {
        while(this.running < this.max && this.queue.length) {
            const task = this.queue.shift();
            task.then(() => {
                this.running--;
                this.next();
            });
            this.running++;
        }
    }
}

const tasks = [task1, task2, task3];
const taskQueue = new TaskQueue(2);

const didTask = new Promise((resolve, reject) => {
    let done = 0, hasErrors = true;
    tasks.forEach(task => {
        taskQueue.push(() => {
            return task().then(() => {
                if(++done === task.length) {
                    resolve();
                }
            }).catch(err => {
                if(!hasErrors) {
                    hasErrors = true;
                    reject();
                }
            });
        });
    });
});

didTask.then(() => {
    callback();
}).then(err => {
    callback(err);
});
```
### 同时暴露两种类型的 API
那么问题来了，如果我需要封装一个库，使用者需要在`callback`和`promise`中灵活切换怎么办呢？让别人一直自己切换就会显得很难用，所以就需要同时暴露`callback`和`promise`的 API ，让使用者传入`callback`的时候使用`callback`，没有传入的时候返回一个`promise`：

```javascript
function asyncDemo(args, callback) {
    return new Promise((resolve, reject) => {
        precess.nextTick(() => {
            // do something 
            // 报错产出 err, 没有则产出 result
            if(err) {
                if(callback) {
                    callback(err);
                }
                return resolve(err);
            }
            if(callback){
                callback(null, result);
            }
            resolve(result);
        });
    });
}
```