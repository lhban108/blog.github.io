
Web Worker
---


### 1. 问题描述：
表格 4000行 25列，共十万条数据，进行计算（包括总和、算平均、最大、最小、计数、中位数、样本方差等），运算加渲染时间共35S左右。并且这个时间段 页面一直处于假死状态，对页面做任何操作都没有反应。

（浏览器有GUI渲染线程与JS引擎线程，这两个线程是互斥的关系。当js有大量计算时，会造成 UI 阻塞，出现界面卡顿、掉帧等情况，严重时会出现页面卡死的情况，俗称 **假死**）。


### 2. Performance分析假死期间的性能表现：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44c0180a16494db2a2f22ae6c97016cf~tplv-k3u1fbpfcp-watermark.image?)


重点从以下三个方面分析：

**1）FPS**

FPS: 表示每秒传输帧数，是分析动画的一个主要性能指标，绿色的长度越长，用户体验越好；反之红色越长，说明卡顿严重

**从图中看到FPS中有一条持续了35s的红线，说明这期间卡顿严重**

**2）火焰图Main**  
Main: 表示主线程运行状况，包括js的计算与执行、css样式计算、Layout布局等等。

展开Main,红色倒三角的为**Long Task**,执行时长50ms就属于长任务，会阻塞页面渲染

**从图中看到计算过程的Long Task执行时间为35.45s, 是造成页面假死的原因**

**3）Summary 统计汇总面板**  
Summary: 表示各指标时间占用统计报表

-   Loading: 加载时间
-   Scripting: js计算时间
-   Rendering: 渲染时间
-   Painting: 绘制时间
-   Other: 其他时间
-   Idle: 浏览器闲置时间

分析结果：**从图中看到计算过程的Long Task执行时间为35.45s, 是造成页面假死的原因（Scripting: 35.9s）**

### 3. 解决办法—— Web Worker

*MDN: Web Worker 为 Web 内容在后台线程中运行脚本提供了一种简单的方法。线程可以执行任务而不干扰用户界面。*

**Web Worker专门处理复杂计算的，从此让前端拥有后端的计算能力**

Web Worker 有以下几点需要注意：

（1）**同源限制** —— 分配给 Worker 线程运行的脚本文件，必须与主线程的脚本文件同源。

（2）**DOM 限制** —— Worker 线程所在的全局对象，与主线程不一样，无法读取主线程所在网页的 DOM 对象，也无法使用`document`、`window`、`parent`这些对象。但是，Worker 线程可以`navigator`对象和`location`对象。

（3）**通信联系** —— Worker 线程和主线程不在同一个上下文环境，它们不能直接通信，必须通过消息完成。

（4）**脚本限制** —— Worker 线程不能执行`alert()`方法和`confirm()`方法，但可以使用 `XMLHttpRequest` 对象发出 AJAX 请求。

（5）**文件限制** —— Worker 线程无法读取本地文件，即不能打开本机的文件系统（`file://`），它所加载的脚本，必须来自网络。

### 4. Web Worker 使用 - 原生态


```html
<!DOCTYPE html>
<html>

<head>
  <meta charset="UTF-8">
  <title>同页面的 Web Worker</title>
</head>

<body>
  <script type="app/worker" id="worker">
    // worker 线程
    // 注意必须指定 <script> 标签的 type 属性是一个浏览器不认识的值

    // 在 worker 线程中，self 和 this 都指向 worker 的全局作用域
    self.onmessage = function(event) { 
      // self 可省略,直接使用 onmessage, 也可以使用 this.onmessage = function(){ ... }
      console.log('worker name:', self.name); // my Worker （self可省略，直接使用 name）
      console.log("2 收到主线程的数据：", event.data); // {data3: '11223aabbcde'}
      
      const { data1, data2 } = event.data;
      
      // 接收到主线程的任务后，在这里处理高密集计算的逻辑
      // 将处理后的数据通过 postMessage 传递给主线程
      const resData = {
        data3: data1 + data2,
      };
      postMessage(resData);
    }

    /*
    // 也可以通过这种方式进行处理监听函数
    self.addEventListener('message', function(event) {
      console.log("1 收到主线程的数据：", event.data); // {data3: '11223aabbcde'}
      const { data1, data2 } = event.data;

      // 接收到主线程的任务后，在这里处理高密集计算的逻辑
      // 将处理后的数据通过 postMessage 传递给主线程
      const resData = {
        data3: data1 + data2,
      };
      postMessage(resData);
    }, false);
    */
  </script>

  <script>
    // 主线程


    // 先将嵌入网页的脚本代码 转成一个二进制对象，然后为这个二进制对象生成 URL
    // 再让 Worker 加载这个 URL。这样就做到了 主线程和 Worker 的代码都在同一个网页上面
    const blob = new Blob([document.querySelector("#worker").textContent]);
    const url = window.URL.createObjectURL(blob);
    const worker = new Worker(url, {
      name: 'my Worker'
    });

    worker.onmessage = function (event) {
      console.log("收到worker的数据：", event.data); // {data1: 11223, data2: 'aabbcde'}

      // 关闭 worker 线程
      worker.terminate();
    };

    const params = {
      data1: 11223,
      data2: 'aabbcde',
    };
    // 可以传递参数给 worker 线程
    worker.postMessage(params);
  </script>
</body>

</html>
```

API 说明：


    主线程：
        
    const worker = new Worker( 'worker.js' , { name: 'myWorker' } );
    `Worker()`构造函数，可以接受两个参数。第一个参数是脚本的网址（必须遵守同源政策），
    该参数是必需的，且只能加载 JS 脚本，否则会报错。第二个参数是配置对象，该对象可选。
    它的一个作用就是指定 Worker 的名称，用来区分多个 Worker 线程。    
    
    -   Worker.onmessage：指定 message 事件的监听函数，发送过来的数据在`Event.data`属性中。
    -   Worker.postMessage()：向 Worker 线程发送消息。
    -   Worker.terminate()：立即终止 Worker 线程。
    -   Worker.onerror：指定 error 事件的监听函数。
    -   Worker.onmessageerror：指定 messageerror 事件的监听函数。发送的数据无法序列化成字符串时，会触发这个事件。
    
    Worker 线程：
    Web Worker 有自己的全局对象，不是主线程的`window`，而是一个专门为 Worker 定制的全局对象。
    因此定义在`window`上面的对象和方法不是全部都可以使用。
    -   self.name： Worker 的名字。该属性只读，由构造函数指定。
    -   self.onmessage：指定`message`事件的监听函数。
    -   self.onmessageerror：指定 messageerror 事件的监听函数。发送的数据无法序列化成字符串时，会触发这个事件。
    -   self.close()：关闭 Worker 线程。
    -   self.postMessage()：向产生这个 Worker 线程发送消息。
    -   self.importScripts()：加载 JS 脚本。
    
    
  

### 5. Web Worker 使用 - vue

vue 推荐使用插件 worker-loader

1、安装 worker-loader
```bash
npm install worker-loader
```

2、编写worker.js （在当前目录中新增 worker.js 文件）

```javascript
// worker.js 

// Worker 线程
onmessage = function (e) {
  // onmessage 获取传入的参数
  let sum = e.data;
  // 进行密集型计算逻辑 
  for (let i = 0; i < 200000; i++) {
    for (let i = 0; i < 10000; i++) {
      sum += Math.random()
    }
  }
  // 将计算的结果传递出去
  postMessage(sum);
}
```

3、通过行内loader 引入 worker.js

```javascript
// 按照worker-loader的要求 引入 Worker.js 的文件
import Worker from "worker-loader!./worker"
```

4、最终代码

```html
<template>
    <div>
        <button @click="makeWorker">开始线程</button>
        <!--在计算时 往input输入值时 没有发生卡顿-->
        <p><input type="text"></p>
    </div>
</template>

<script>
	// 按照worker-loader的要求 引入 Worker.js 的文件
    import Worker from "worker-loader!./worker.js";

    export default {
        methods: {
            makeWorker() {
                // 获取计算开始的时间
                let start = performance.now();
                // 新建一个线程
                let worker = new Worker();
                // 线程之间通过postMessage进行通信
                worker.postMessage(0);
                // 监听message事件
                worker.addEventListener("message", (e) => {
                    // 关闭线程
                    worker.terminate();
                    // 获取计算结束的时间
                    let end = performance.now();
                    // 得到总的计算时间
                    let durationTime = end - start;
                    console.log('计算结果:', e.data);
                    console.log(`代码执行了 ${durationTime} 毫秒`);
                });
            }
        },
    }
</script>
```







> [# 一文彻底了解Web Worker，十万条数据都是弟弟](https://juejin.cn/post/7137728629986820126)

> [# Web Worker 使用教程[阮一峰]](https://www.ruanyifeng.com/blog/2018/07/web-worker.html)

> [# 一文彻底学会使用web worker](https://juejin.cn/post/7139718200177983524)
  
### 6. Web Worker 使用场景

（1）加密 —— 端到端加密的使用场景越来越多，但是加密有时候会非常的耗时，特别是当需要经常加密大量数据的时候（比如，发往服务器前数据加密）。这是一个使用Web Worker 的绝佳场景，因为不需要访问DOM 或者利用其他魔法 —— 它只是纯粹使用算法进行计算而已。
（2）预取数据 —— 为了优化网站或者网络应用及减少数据加载时间，可以使用Web Worker 来提前加载部分数据以备不时之需。并且这种情况下不会影响程序的使用体验。
（3）射线追踪 —— 射线追踪是一项通过追踪光线的路径作为像素来生成图片的渲染技术。这些计算逻辑可以放在 Web Worker 中以避免阻塞 UI 线程。
（4）拼写检查 —— 一个基本的拼写检测器是这样工作的－程序会读取一个包含拼写正确的单词列表的字典文件。字典会被解析成一个搜索树以加快实际的文本搜索。当检查器检查一个单词的时候，程序会在预构建搜索树中进行检索。如果在树中没有检索到，则会通过替换可选单词并测试单词是否可用-如果是用户所需的单词来用户提供替代的拼写。这个检索过程中的所有工作都可以交由 Web Worker 来完成，这样用户就只需输入单词和语句而不会阻塞 UI，与此同时 worker 会处理所有的搜索和供给建议。


