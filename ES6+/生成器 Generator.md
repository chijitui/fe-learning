在新的 ES 标准中，**生成器（Generator）** 几乎是一个完全崭新的函数类型，它能生成一组值得序列，但每个值的生成是基于每次请求，并不同于标准函数那样立即生成。

这样特殊的性质，也使得它成为了我们解决 JS 异步编程的一个利器。

### 生成器的遍历
要遍历生成器很简单，只需要用`for...of`函数就可以取出每一步从生成器中返回的值。举个例子：
```javascript
function* AnimalGenerator() {
    yield 'dog';
    yield 'cat';
    yield 'fish';
}

for(let animal of AnimalGenerator()) {
    console.log(animal);
}
// => dog cat fish
```

### 生成器嵌套
生成器嵌套的方式，可以将一个生成器委托给另外一个生成器，在执行到某个节点的时候，返回该生成器中的值：
```javascript
function* AllGenerator() {
    yield 'human';
    yield* AnimalGenerator();
    yield 'AI';
}

for(let item of AllGenerator()) {
    console.log(item);
}
// => human dog cat fish AI
```

### 迭代器
在调用生成器之后，会创建一个 **迭代器（Iterator）** 并返回出来，而利用迭代器就可以逐步返回生成器中的值：
```javascript
const animalIterator = AnimalGenerator();

const animal1 = animalIterator.next();
const animal2 = animalIterator.next();
const animal3 = animalIterator.next();
const animal4 = animalIterator.next();

console.log(animal1, animal2, animal3, animal4);
// => {value: "dog", done: false} {value: "cat", done: false} {value: "fish", done: false} {value: undefined, done: true}
```
在每次执行迭代器`next`方法的时候，返回一个对象，对象包含两个属性：
- **value**：生成器的返回值，生成结束之后返回`undefined`;
- **done**：表示生成器是否生成完毕，返回`false`或`true`。

除此之外，迭代器的`next`还可以接收一个参数作为上一次`yield`的返回值：
```javascript
function* Next(x) {
    const y = 3 * yield (x + 1);
    const z = yield (y / 10);
    yield (x + y + z);
}

const next = Next(1);

const result1 = next(); 
// => value: 2
const result2 = next(10); 
// => value: 3
const result3 = next(12); 
// => value: 43
```

这结果看起来有点绕，但是捋一捋就会发现很有意思：
- 第一次调用`next`，返回值是第一个`yield`，因此就是`x+1`，即`2`；
- 第二次调用`next(10)`，传入的值替代了第一个`yield`的返回值，所以`y`的值为`3*10`，即`30`，返回第二个`yield`就是`30/10`，即`3`；
- 第三次调用`next(12)`，传入的值替代了第二个`yield`，所以`z`的返回值为`12`，加上之前生成器缓存的`x=1`和`y=30`，最终返回`x+y+z`的值就是`43`。

### 生命周期
看了这些生成器的基本用法，肯定会对它的内部原理很好奇。概括一下，就是直接调用一个生成器不会直接执行，而是会创建一个新的迭代器返回出来，通过迭代器我们才能对其进行求值。生成器每生成一个值都会自己挂起执行，然后等待下一个请求。

**在某种当面来说，生成器的工作更像是一个在状态中运动的状态机。**

![生成器生命周期](https://upload-images.jianshu.io/upload_images/5236403-790343f41772007d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上图所示，生成器的生命周期主要分为四个阶段：
- **Start**：创建一个生成器以后，不执行任何代码，然后挂起；
- **Exucuting**：执行生成器的代码，通过迭代器的`next`方法调用，只要还有可执行代码，生成器都会进入这个状态；
- **Yield**：生成器每执行一次，都到`yield`表达式处包裹一个对象返回，然后挂起等待执行；
- **Completed**：如果代码全部执行完毕，或者遇到`return`，就执行完毕。

从生命周期我们可以看到，在整个周期里，只要我们取得生成器的控制权，它的执行上下文就会一直保存，而不会想标准函数一样退出后就销毁。
