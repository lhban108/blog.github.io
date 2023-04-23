# JS 手写题 —— 字符串、Object相关

### 1.字符串——的大小写切换

例: 'aaA23BsP' -> 'AAa23bSp'

```JavaScript
function caseConvert(str) {
  if (!str) return str

  return str.replace(/([a-z]*)([A-Z]*)/g, (m, s1, s2) => {
    return `${s1.toUpperCase()}${s2.toLowerCase()}`;
  })
}

caseConvert('aaA23BsP'); // 'AAa23bSp'
```

### 2.解析 Url 参数为对象

### 3.字符串——下划线转驼峰

### 4.手写 JSON.stringfy

### 5.实现一个简易版模板引擎

### 6.列表转树形结构 / 树形结构转列表

### 7.手写 new()、Object.create()

### 8.请实现 DOM2JSON 一个函数，可以把一个 DOM 节点输出 JSON 的格式

