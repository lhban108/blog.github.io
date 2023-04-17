# JS 基础手写题

## 1. 浅/深拷贝

浅拷贝：

- Object.assign()
- 函数库 lodash 的 lodash.clone 方法
- 展开运算符 …
- Array.prototype.concat()
- Array.prototype.slice()

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
    if (Object.hasOwnProperty.call(obj, key)) {
    // if (obj.hasOwnProperty(key)) {
      cloneObj[key] = deepClone(obj[key], hash)
    }
  }
  return cloneObj;
}

```

## 2. 防抖/节流
