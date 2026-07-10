# ES6 Iterator 与 for...of 核心考点（32、33）
## 32. Iterator 接口 + for...of + 默认部署 Iterator 的数据结构
### 一、Iterator 是什么
迭代器（遍历器），是一种统一**遍历数据**的接口规范，作用：
1. 为不同数据结构提供统一遍历方式
2. 供 `for...of`、扩展运算符 `...`、`Array.from`、解构赋值、`yield*` 消费

### 二、迭代器协议
对象必须拥有 `Symbol.iterator` 方法，调用后返回**迭代器对象**；
迭代器对象必须拥有 `.next()` 方法，每次调用返回结构：
```js
{ value: 当前遍历值, done: 布尔值 }
// done=true 代表遍历结束，value 为 undefined
```

### 三、原生自带默认 Iterator 接口（可直接 for...of）
这些类型原型上内置了 `Symbol.iterator`，开箱即用：
1. Array 数组
2. Set / Map
3. String 字符串（第33题单独讲）
4. TypedArray 类型数组
5. arguments 类数组
6. NodeList DOM节点集合

### 四、for...of 原理
`for...of` 会自动调用目标的 `Symbol.iterator()` 获取迭代器，循环执行 `.next()`，直到 `done: true` 终止。
```js
const arr = [10,20,30];
// 底层等价
const it = arr[Symbol.iterator]();
it.next(); // {value:10,done:false}
it.next(); // {value:20,done:false}
it.next(); // {value:30,done:false}
it.next(); // {value:undefined,done:true}

// 简化写法
for(const v of arr){
  console.log(v)
}
```

### 五、无默认 Iterator 的结构（不能直接 for...of）
普通对象 `{}` 没有部署 `Symbol.iterator`，直接 `for...of` 会报错；
如需遍历对象，手动部署迭代器或用 `Object.keys/values/entries` 中转。

### 六、会自动调用 Iterator 的语法
- `for...of` 循环
- 扩展运算符 `[...data]`
- `Array.from(data)`
- 解构赋值 `const [a,b] = data`
- `yield* data`（生成器委托）
- `Set/Map` 构造函数传参 `new Set(data)`

---

## 33. 字符串的 Iterator 接口
### 1. 字符串原生自带 Symbol.iterator
字符串被视为字符序列，可直接 `for...of` 遍历每一个字符，**完美识别 Unicode 四字节字符（emoji、生僻汉字）**，弥补普通 `for` 循环下标缺陷。
```js
const str = 'ab😀c';
// for...of 正常拆分四字节字符
for(const char of str){
  console.log(char) // a b 😀 c
}
```

### 2. 对比普通下标遍历的缺陷
```js
const str = '😀';
str[0]; // 高代理项，乱码
str[1]; // 低代理项，乱码
// 普通for无法正确识别单个emoji
```

### 3. 手动获取字符串迭代器
```js
const str = 'hi';
const it = str[Symbol.iterator]();
it.next() // {value:'h', done:false}
it.next() // {value:'i', done:false}
it.next() // {value:undefined, done:true}
```

### 4. 相关应用
- 字符串展开：`[...'hello'] // ['h','e','l','l','o']`
- 拆分含特殊Unicode字符的字符串，比 `split('')` 更可靠

### 5. 补充：字符串 Iterator 特性
遍历单位是**Unicode码点**，而非UTF-16码元，是ES6专门为解决生僻字符遍历问题设计的迭代器实现。

---

# 两道题核心总结
1. 32：Iterator 是统一遍历接口，原生6类数据自带默认迭代器，`for...of` 基于迭代器实现；普通对象无默认迭代器。
2. 33：字符串内置迭代器，按Unicode码点遍历，能正确处理四字节字符，是字符串专属迭代器实现。