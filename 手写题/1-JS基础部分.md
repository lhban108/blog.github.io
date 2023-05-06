# JS 基础手写题 —— 函数相关

### 1. 浅/深拷贝

浅拷贝：

- `Object.assign()`
- 函数库 lodash 的 `lodash.clone` 方法
- 展开运算符 …
- `Array.prototype.concat()`
- `Array.prototype.slice()`

```JavaScript
// 浅拷贝实现过程
let obj = {
 name: 'Yvette',
 age: 18,
 hobbies: ['reading', 'photography']
}
// 浅拷贝实现方式一: Object.assign()
let obj2 = Object.assign({}, obj);

// 浅拷贝实现方式二: 函数库lodash的_.clone方法
let lodash = require('lodash');
let obj3 = lodash.clone(obj);

// 浅拷贝实现方式三: 展开运算符...
let obj4 = {...obj};

// 浅拷贝实现方式四: Array.prototype.concat()
let arr1 = [1, 3, { username: 'kobe' }];
let arr2 = arr1.concat();  

// 浅拷贝实现方式五: Array.prototype.concat()
let arr3 = [1, 3, { username: 'kobe' }];
let arr4 = arr3.slice();
```

深拷贝：

- JSON.parse(JSON.stringify(obj))
- 函数库 lodash 的 lodash.cloneDeep 方法
- jQuery.extend()方法
- 手写递归方法

```JavaScript
// 深拷贝实现方式一: JSON.parse(JSON.stringify(obj))
// 缺点：
// 1. 对象属性是函数时，无法拷贝
// 2. 原型链上的属性无法拷贝
// 3. 不能正确处理 Date、RegExp 类型的数据
// 4. 会忽略 Symbol 和 undefined

// 深拷贝实现方式二: 函数库 lodash 的 lodash.cloneDeep 方法

// 深拷贝实现方式三: jQuery.extend()方法
const $ = require('jquery');
const obj1 = {
 a: 1,
 b: { f: { g: 1 } },
 c: [1, 2, 3],
};
const obj2 = $.extend(true, {}, obj1); //第一个参数为true, 就是深拷贝

// 深拷贝实现方式四: 实现一个 deepClone 函数
// 步骤：
// 1、如果是基本数据类型，直接返回
// 2、如果是 RegExp 或者 Date 类型，返回对应类型
// 3、如果是复杂数据类型，递归
// 4、考虑循环引用的问题

function deepClone(obj, hash = new WeakMap()) {
  // 如果是基本数据类型 或者 null，直接返回
  if (obj === null || typeof obj !== 'object') {
    return obj;
  }

  // 如果是 RegExp 或者 Date 类型，返回对应类型
  if (obj instanceof Date) {
    return new Date(obj);
  }
  if (obj instanceof RegExp) {
    return new RegExp(obj);
  }

  // 考虑循环引用的问题
  if (hash.has(obj)) {
    return hash.get(obj);
  }

  // 递归
  // obj.__proto__.constructor === obj.constructor
  const cloneObj = new obj.constructor();
  hash.set(obj, cloneObj)
  for (key in obj) {
    // if (obj.hasOwnProperty(key)) {
    if (Object.hasOwnProperty.call(obj, key)) {
      cloneObj[key] = deepClone(obj[key], hash)
    }
  }
  return cloneObj;
}

```

### 2. 防抖/节流

防抖(debounce)：触发事件后，在 n 秒后只能执行一次，如果在 n 秒内又触发了事件，则会重新计算函数的执行时间
防抖使用场景：

- 1. 搜索框输入查询 —— 如果用户一直在输入中，等用户停止输入的时候，再调用，设置一个合适的时间间隔，有效减轻服务端压力
- 2. 表单验证 —— 手机号、邮箱格式的输入验证检测
- 3. 按钮提交事件
- 4. 浏览器窗口缩放 —— resize事件，窗口停止改变大小之后再重新计算布局，防止重复渲染
  
```JavaScript
// 防抖函数 debounce
function debounce(fn, delay = 500) {
  let timeout = null;
  return function() {
    clearTimeout(timeout);
    timeout = setTimeout(() => {
      fn.apply(this, arguments);
      timeout = null;
    }, delay)
  }
}

// 验证
const inputDom = docement.getElementById('input1');
inputDom.addEventListener('keyup', debounce(functuon(){
  console.log(inputDom.value);
}, 600))
```

节流(throttle)：触发事件后的 n 秒内不再触发该事件

节流使用场景：

- 1. 按钮点击事件 —— 高频点击提交，表单重复提交
- 2. 滚动加载，加载更多活滚动到底部监听
- 3. 拖拽事件

```JavaScript
// 节流函数1: throttle
function throttle(fn, delay = 500) {
  let flag = false;
  return function(...args) {
    if (flag) return
    flag = true;
    setTimeout(() => {
      fn.apply(this, args);
      flag = false;
    }, delay)
  }
}

// 节流函数2: throttle
function throttle(fn, delay = 500) {
  let timer = null;
  return function(...args) {
    if (!timer) {
      timer = setTimeout(() => {
        fn.apply(this, args);
        timer = null;
      }, delay)
    }
  }
}

// 验证
let div1 = docuemnt.getElementById('div1');
div1.addEventListener('drag', throttle(function(e) {
  console.log(e.offsetX, e.offsetY);
}, 200))
```

应用场景一：统计按钮一秒的点击次数

```HTML
<body>
  <button id="butn1">butn</button>
</body>
<script>
  const butn1 = document.getElementById('butn1');
  butn1.onclick = debounce(function() {
    console.log('点击一次');
  }, 1000)

  function debounce(fn, delay) {
    let num = 0;
    let timer = null;
    return function(...args) {
      num++;
      fn.apply(this, args);
      if (timer) return
      timer = setTimeout(() => {
        console.log('一共点击了' + num + '次');
        num = 0;
        timer = null;
      }, delay)
    }
  }
</script>
```

### 3. 函数柯里化

```JavaScript
var curry = function(fn) {
  return function curried(...args) {
    if (args.length === fn.length) return fn(...args);
    return (...arg) => curried(...args, ...arg);
  };
};

// 验证
const fn = function sum(a, b, c) { return a + b + c; };
const curriedSum = curry(fn);
curriedSum(1)(2)(3); // 6
curriedSum(1, 2)(3); // 6
curriedSum(1)(2, 3); // 6

```

### 4.Call/Bind/Apply

### 6. sleep —— 实现一个函数，n秒后执行函数func

### 7.手写一个 菲波那切数列

### 8.实现一个sum函数

### 9.手写 instanceof()、获取JS类型函数

```JavaScript
// instanceof
function instanceOf(left, right) {
  let proto = left.__proto__;
  const prototype = right.constructor;
  while(true) {
    if (proto === null) return false;
    if (proto === prototype) return true;
    proto = proto.__proto__;
  }
}
```

### 10.手写一个JSONP

```JavaScript
// 解题思路：
// 1. 如果是简单数据类型，String(obj)
// 2. 如果是数组，取数组的值，加上 []
// 3. 如果是对象，取键值对，加上 {}
function jsonStringify(obj) {
  const type = typeof obj;
  if (type !== 'object' || obj === null) {
    if (/string|undefined|function/.test(type)) {
      obj = '"' + obj + '"';
    }
    return String(obj);
  } else {
    const json = [];
    const isArray = obj && Array.isArray(obj);
    for (const key in obj) {
      let val = obj[key];
      val = jsonStringify(val);
      json.push((isArray ? '' : '"' + key + '":') + String(val));
    }
    return (isArray ? '[' : '{' ) + String(json) + (isArray ? ']' : '}');
  }
}
```

> [42+高频js手写题及答案](https://mp.weixin.qq.com/s/CIDYqlXxq4aWY6UizjfZvg)
