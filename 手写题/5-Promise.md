# JS 手写题 —— Promise

### 1. Promise 异步调度函数 - 1 （头条题）

> [Github 实现有并行限制的 Promise 调度器](https://github.com/Sunny-117/js-challenges/issues/149)

题目描述：JS实现一个带并发限制的异步调度器Scheduler，保证同时运行的任务最多有两个。完善下面代码的Scheduler类，使以下程序能够正常输出。

```JavaScript
class Scheduler {
  add(promiseCreator) { ... }
  // ...
}
   
const timeout = time => new Promise(resolve => {
  setTimeout(resolve, time);
})
  
const scheduler = new Scheduler();
  
const addTask = (time, order) => {
  scheduler.add(() => timeout(time).then(() => {
    console.log(order);
  }))
}

addTask(1000, '1');
addTask(500, '2');
addTask(300, '3');
addTask(400, '4');

// output: 2 3 1 4

// 整个的完整执行流程：
// (1) 1、2两个任务开始执行
// (2) 500ms时，2任务执行完毕，输出2，任务3开始执行
// (3) 800ms时，3任务执行完毕，输出3，任务4开始执行
// (4) 1000ms时，1任务执行完毕，输出1
// (5) 1200ms时，4任务执行完毕，输出4
```

题解：

```JavaScript
class Scheduler {
  constructor(max) {
    // 最大可并发任务数
    this.max = max;
    // 当前并发任务数
    this.count = 0;
    // 阻塞的任务队列
    this.queue = [];
  }

  async add(fn) {
    if (this.count >= this.max) {
      // 若当前正在执行的任务，达到最大容量max
      // 阻塞在此处，等待前面的任务执行完毕后将resolve弹出并执行
      await new Promise(resolve => {
        this.queue.push(resolve);
      });
    }
    // 当前并发任务数 +1
    this.count++;
    // 使用await执行此函数
    await fn();
    // 执行完毕，当前并发任务数 -1
    this.count--;
    // 若队列中有值，将其resolve弹出，并执行
    // 以便阻塞的任务，可以正常执行
    this.queue.length && this.queue.shift()();
  }
}

```

### 2. Promise 异步调度函数 - 2 （Promise 对象池）

> [Leetcode 2636. Promise 对象池](https://leetcode.cn/problems/promise-pool/)

请你编写一个异步函数 `promisePool` ，它接收一个异步函数数组 `functions` 和 **池限制** n。它应该返回一个 promise 对象，当所有输入函数都执行完毕后，promise 对象就执行完毕。

**池限制** 定义是一次可以挂起的最多 promise 对象的数量。`promisePool` 应该开始执行尽可能多的函数，并在旧的 promise 执行完毕后继续执行新函数。`promisePool` 应该先执行 `functions[i]`，再执行 `functions[i + 1]`，然后执行 `functions[i + 2]`，等等。当最后一个 promise 执行完毕时，`promisePool` 也应该执行完毕。

例如，如果 `n = 1` , `promisePool` 在序列中每次执行一个函数。然而，如果 `n = 2` ，它首先执行两个函数。当两个函数中的任何一个执行完毕后，再执行第三个函数(如果它是可用的)，依此类推，直到没有函数要执行为止。

你可以假设所有的 `functions` 都不会被拒绝。对于 `promisePool` 来说，返回一个可以解析任何值的 promise 都是可以接受的。

**示例1：**

```text
输入：
functions = [
  () => new Promise(res => setTimeout(res, 300)),
  () => new Promise(res => setTimeout(res, 400)),
  () => new Promise(res => setTimeout(res, 200))
]
n = 2
输出：[[300,400,500],500]
解释
传递了三个函数。它们的睡眠时间分别为 300ms、 400ms 和 200ms。
在 t=0 时，执行前两个函数。池大小限制达到 2。
当 t=300 时，第一个函数执行完毕后，执行第3个函数。池大小为 2。
在 t=400 时，第二个函数执行完毕后。没有什么可执行的了。池大小为 1。
在 t=500 时，第三个函数执行完毕后。池大小为 0，因此返回的 promise 也执行完成。
```

**示例2：**

```text
输入：
functions = [
  () => new Promise(res => setTimeout(res, 300)),
  () => new Promise(res => setTimeout(res, 400)),
  () => new Promise(res => setTimeout(res, 200))
]
n = 5
输出：[[300,400,200],400]
解释：
在 t=0 时，所有3个函数都被执行。池的限制大小 5 永远不会满足。
在 t=200 时，第三个函数执行完毕后。池大小为 2。
在 t=300 时，第一个函数执行完毕后。池大小为 1。
在 t=400 时，第二个函数执行完毕后。池大小为 0，因此返回的 promise 也执行完成。
```

**示例3：**

```text
输入：
functions = [
  () => new Promise(res => setTimeout(res, 300)),
  () => new Promise(res => setTimeout(res, 400)),
  () => new Promise(res => setTimeout(res, 200))
]
n = 1
输出：[[300,700,900],900]
解释：
在 t=0 时，执行第一个函数。池大小为1。
当 t=300 时，第一个函数执行完毕后，执行第二个函数。池大小为 1。
当 t=700 时，第二个函数执行完毕后，执行第三个函数。池大小为 1。
在 t=900 时，第三个函数执行完毕后。池大小为 0，因此返回的 Promise 也执行完成。
```

**题解：**

```JavaScript
var promisePool = async function(functions, n = 2) {
  const resArr = [];
  const arr = [...functions];
  const curSet = new Set();

  while(arr.length) {
    if (curSet.size < n) {
      const item = arr.shift();
      const pro = item().then((res) => {
        resArr.push(res);
        curSet.delete(pro);
      });
      curSet.add(pro);
    }
    if (curSet.size === n) {
      await Promise.race(curSet);
    }
  }
  await Promise.allSettled(curSet);

  return resArr;
}
```

### 3. Promise 异步调度函数 - 3 (Promise 串行与并行)

> [一道赋值面试题引发的思考3【并发数控制】](https://libin1991.github.io/2019/02/06/%E4%B8%80%E9%81%93%E8%B5%8B%E5%80%BC%E9%9D%A2%E8%AF%95%E9%A2%98%E5%BC%95%E5%8F%91%E7%9A%84%E6%80%9D%E8%80%833%E3%80%90%E5%B9%B6%E5%8F%91%E6%95%B0%E6%8E%A7%E5%88%B6%E3%80%91/)

题目描述：实现 mergePromise 函数，把传进去的函数数组按顺序先后执行，并且把返回的数据先后放到数组 data 中。

```JavaScript
// 题目描述：
const timeout = ms => new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve();
  }, ms);
});

const ajax1 = () => timeout(5000).then(() => {
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
  // ...
  // 在这里实现你的代码

};

mergePromise([ajax1, ajax2, ajax3]).then(data => {
  console.log('done');
  console.log(data); // data 为 [1, 2, 3]
});

// 要求分别输出
// 1
// 2
// 3
// done
// [1, 2, 3]
```

分析：
timeout 是一个函数，这个函数执行后返回一个promise实例。  
ajax1、ajax2、ajax3 都是函数，不过这些函数有一些特点，执行后都会会返回一个 新的promise实例。  
我们需要让 ajax1 执行完之后 再执行 ajax2，再执行 ajax3，而不能让他们同时执行

```JavaScript
// 错误解法(并发) —— 使用 async await
const mergePromise = async (ajaxArray) => {
  // 记录开始时间
  const timeStart = Date.now();

  const arr = [];

  // 并发执行
  const proList = ajaxArray.map(item => item());

  for (const pro of proList) {
    arr.push(await pro);
  }

  // 获取 执行用时
  console.log(Date.now() - timeStart); // => 5000
  // 打印的顺序是 2 3 1 , 不符合题目要求
  // 错误原因：3个 ajax 属于并发执行，不是按照顺序串行的，执行共用时 5000 ms，

  return arr;
};
```

```JavaScript
// 正确解法(串行) - 使用 Promise.resolve()
const mergePromise = (ajaxArray) => {
  // 记录开始时间
  const timeStart = Date.now();

  const data = [];

  // Promise.resolve方法调用时不带参数，直接返回一个resolved状态的 Promise 对象
  const sequencePromise = Promise.resolve();

  // Tips: forEach 中不能使用 async 和 await
  // Array.prototype.forEach不是为异步代码设计的,它执行后会立即返回，不会等待await
  ajaxArray.forEach((item) => {
    // 第一次的 then 方法用来执行数组中的每个函数，
    // 第二次的 then 方法接受数组中的函数执行后返回的结果，
    // 并把结果添加到 data 中，然后把 data 返回。
    sequencePromise = sequencePromise.then(item).then((res) => {
      data.push(res);
      return data;
    })
  })

  // 获取 执行用时
  console.log(Date.now() - timeStart); // => 8000
  // 因为 sequencePromise 返回一个Promise对象，并且每次执行完后获取一个新的Promise对象
  // 即 数组中的3个 Promise对象是 按顺序串行的，所以执行时间累加 5000+1000+2000

  // 遍历结束后，返回一个 Promise(也就是sequence)，他的 [[PromiseValue]] 值就是 data
  // 而 data（保存数组中的函数执行后的结果）也会作为参数，传入下次调用的 then 方法中。
  return sequencePromise;
}
```

### 4. Promise 异步调度函数 - 4 (Promise 指定并发调度个数)

题目描述：请实现如下函数，可以批量请求数据，所有 URL 地址在 urls 参数中，可以通过 max参数控制请求的最大并发数。

```JavaScript
function sendRequest(urls, max) {
  // 在这里实现你的代码
}

var urls = [
  'https://www.kkkk1000.com/images/getImgData/getImgDatadata.jpg', 
  'https://www.kkkk1000.com/images/getImgData/gray.gif', 
  'https://www.kkkk1000.com/images/getImgData/Particle.gif', 
  'https://www.kkkk1000.com/images/getImgData/arithmetic.png', 
  'https://www.kkkk1000.com/images/getImgData/arithmetic2.gif', 
  'https://www.kkkk1000.com/images/getImgData/getImgDataError.jpg', 
  'https://www.kkkk1000.com/images/getImgData/arithmetic.gif', 
  'https://www.kkkk1000.com/images/wxQrCode2.png'
];

function loadImg(url) {
  return new Promise((resolve, reject) => {
    const img = new Image()
    img.onload = function () {
      console.log('一张图片加载完成');
      resolve();
    }
    img.onerror = reject
    img.src = url
  })
};
```

题解：

```JavaScript
// 解法一：递归
function sendRequest(urls, max) {
  let num = 0;

  function request() {
    num = num + 1;
    loadImg(urls.shift()).then(() => {
      num = num - 1;
    }).then(() => {
      // 2. 当某个任务加载完成后，进行递归调用
      if (num < max && urls && urls.length) {
        request();
      }
    })
  }
  for (let i = 0; i < max; i++) {
    // 1. 首先启动 max 个并行的任务
    request();
  }
}

// 解法二：async await 阻塞
function sendRequest(urls, max) {
  let num = 0;
  const len = urls.length;
  const lockArr = [];

  async function request() {
    // 如果达到最大并发数，利用 await 进行阻塞
    if (num >= max) {
      await new Promise((resolve, reject) => {
        lockArr.push(resolve);
      })
    }
    if (urls.length) {
      num++;
      await loadImg(urls.shift());
      num--;

      // 当某个任务执行完，释放一个阻塞的任务
      lockArr.length && lockArr.shift()();
    }
  }

  for (let i = 0; i < len; i++) {
    request();
  }

}

```

















