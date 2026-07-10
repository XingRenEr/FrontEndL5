# JS let 完整详解
## 一、核心特点
`let` ES6 新增声明变量，解决 `var` 缺陷：**块级作用域、不存在变量提升、不能重复声明、暂时性死区**

## 1. 块级作用域（最关键）
`{}`、`if`、`for`、`while`、`switch` 大括号内部为独立作用域，外部访问不到
```js
if (true) {
  let a = 10;
  console.log(a); // 10
}
console.log(a); // 报错：a is not defined
```
### for 循环经典场景
`let` 每次循环生成独立变量，不会泄露到外部
```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// 输出 0 1 2

// 换成 var 会全部输出 3
```

## 2. 暂时性死区 TDZ（无变量提升）
在当前块内，**声明前不能使用**，不存在 var 那种提前访问为 `undefined` 的提升
```js
console.log(x); // 报错 Cannot access 'x' before initialization
let x = 5;
```
只要进入当前块，变量就被绑定，声明前整块都是死区：
```js
let num = 10;
if (true) {
  console.log(num); // 报错，本块let num屏蔽外层
  let num = 20;
}
```

## 3. 禁止同一作用域重复声明
```js
let a = 1;
let a = 2; // 语法报错

// 跨作用域不冲突
if(true){
  let a = 3; // 正常
}
```

## 4. 不存在全局污染
全局 `let` 不会挂载到 `window` 对象
```js
var v = 100;
let l = 200;
console.log(window.v); // 100
console.log(window.l); // undefined
```

## 5. 可修改、可先声明后赋值
```js
let name;
name = "小明";
name = "小红"; // 允许重新赋值
```

## let / var / const 简单对比
| 特性           | var                     | let                       | const          |
| -------------- | ----------------------- | ------------------------- | -------------- |
| 作用域         | 函数级                  | 块级                      | 块级           |
| 变量提升       | 有，可提前访问undefined | 有提升但TDZ，不能提前访问 | 同let          |
| 重复声明       | 允许                    | 禁止                      | 禁止           |
| 全局挂载window | 是                      | 否                        | 否             |
| 重新赋值       | 可以                    | 可以                      | 不允许（常量） |

## 常见坑
1. 块外访问 let 变量直接报错，不像 var 变成全局
2. 函数、if、for 块隔离变量，循环定时器优先用 let
3. 同一大括号内不能重复 let 同名变量
4. 不要在 TDZ 内读取变量

## 使用场景
- 循环计数 `for(let i=0...)`
- 块内临时变量、会修改的值
- 替代 var，现代开发优先使用 let / const