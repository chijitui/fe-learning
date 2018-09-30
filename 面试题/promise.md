### 题目一：
来源：阿里校招
```javascript
// 实现 mergePromise 函数，把传进去的数组顺序先后执行，并且把返回的数据先后放到数组（data）中。
const timeout = ms => new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve();
  }, ms);
});

const ajax1 = () => timeout(2000).then(() => {
  console.log('1');
  return 1;
});

const ajax2 = () => timeout(1000).then(() => {
  console.log('2');
  return 2;
});

const ajax3 = () => timeout(2000).then(() => {
  console.log('3');
  return 3;
});

const mergePromise = ajaxArray => {
  // 在这里实现你的代码
};

mergePromise([ajax1, ajax2, ajax3]).then(data => {
  console.log('done');
  console.log(data); // data 为 [1, 2, 3]
});

// 分别输出
// 1
// 2
// 3
// done
// [1, 2, 3]
```

解析1：这是一个考察用控制异步代码顺序执行的题目，工作中会用到很多。关于这一类的知识点可以查看 [Promise 异步控制流](https://github.com/chijitui/fe-learning/blob/master/ES6+/Promise异步控制流.md)。
- 用 Promise 实现：
```javascript

const timeout = ms => new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve();
  }, ms);
});
const ajax1 = () => timeout(2000).then(() => {
  console.log('1');
  return 1;
});
const ajax2 = () => timeout(1000).then(() => {
  console.log('2');
  return 2;
});

const ajax3 = () => timeout(2000).then(() => {
  console.log('3');
  return 3;
});

const mergePromise = ajaxArray => {
  const data = [];
  const didTask = ajaxArray.reduce((prev, task) => {
    return prev.then((value) => {
      if(value !== '__promise__head') {
        data.push(value);
      }
      return task();
    });
  }, Promise.resolve('__promise__head')).then((value) => {
    data.push(value);
    return data;
  });
  return didTask;
};

mergePromise([ajax1, ajax2, ajax3]).then(data => {
  console.log('done');
  console.log(data); // data 为 [1, 2, 3]
});

```

- 用 Async/Await 实现：
```javascript
const timeout = ms => new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve();
  }, ms);
});

const ajax1 = () => timeout(2000).then(() => {
  console.log('1');
  return 1;
});

const ajax2 = () => timeout(1000).then(() => {
  console.log('2');
  return 2;
});

const ajax3 = () => timeout(2000).then(() => {
  console.log('3');
  return 3;
});

const mergePromise = ajaxArray => {
  return new Promise(async (resolve, reject) => {
    const data = [];
    for(let i = 0; i < ajaxArray.length; i++) {
      const value = await ajaxArray[i]();
      data.push(value);
    }
    resolve(data);
  });
};

mergePromise([ajax1, ajax2, ajax3]).then(data => {
  console.log('done');
  console.log(data); // data 为 [1, 2, 3]
});
```
