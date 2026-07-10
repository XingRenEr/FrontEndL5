# JS 四种 for 循环完整讲解
## 一、普通 for 循环（标准计数循环）
### 语法
```js
for(初始化; 条件判断; 自增/自减){
  循环体
}
```
执行流程：
1. 初始化（只执行1次）
2. 判断条件，true 执行循环体；false 结束
3. 执行自增
4. 回到步骤2

### 示例
```js
// 输出 0~4
for(let i = 0; i < 5; i++){
  console.log(i)
}

// 倒序 5~1
for(let i = 5; i >= 1; i--){
  console.log(i)
}
```
### 细节
- `let`：块级作用域，循环内单独变量，推荐使用
- `var`：函数作用域，循环结束后 i 仍存在，易出bug
- 三个表达式均可省略（死循环 `for(;;){}`）

## 二、for in 遍历对象键 / 数组下标
作用：遍历**可枚举属性名**（字符串类型）
### 对象遍历
```js
const obj = {name:"张三", age:18}
for(let key in obj){
  console.log(key, obj[key]) 
  // name 张三  / age 18
}
```
### 数组遍历（不推荐）
```js
const arr = [10,20,30]
for(let index in arr){
  console.log(index, arr[index])
  // index 是字符串 "0","1","2"
}
```
### 缺点
1. 会遍历原型链上的属性
2. 数组遍历下标为字符串，无法直接运算
3. 不保证数字数组顺序

## 三、for of（ES6，遍历可迭代对象：数组、字符串、Map、Set）
### 遍历数组（最常用）
```js
const arr = [1,2,3]
for(let item of arr){
  console.log(item) // 1 2 3
}
```
### 获取下标 + 值：Array.entries()
```js
const arr = ["a","b"]
for(let [idx, val] of arr.entries()){
  console.log(idx, val)
}
```
### 遍历字符串
```js
for(let s of "hello"){
  console.log(s) // h e l l o
}
```
### 遍历 Map / Set
```js
// Set
const set = new Set([1,2,3])
for(let v of set) console.log(v)

// Map
const map = new Map([["name","李四"],["age",20]])
for(let [k,v] of map) console.log(k,v)
```
### 限制
**不能直接遍历普通对象**（普通对象不可迭代）

## 四、for await of 异步循环（遍历异步迭代器）
用于异步数据流、读取文件、接口分页等
```js
async function loop(){
  const arr = [Promise.resolve(1), Promise.resolve(2)]
  for await(let res of arr){
    console.log(res)
  }
}
loop()
```

## 五、循环控制关键字
1. `break`：直接终止整个循环
```js
for(let i=0;i<10;i++){
  if(i === 3) break
  console.log(i) // 0 1 2
}
```
2. `continue`：跳过本次，直接进入下一轮
```js
for(let i=0;i<5;i++){
  if(i === 2) continue
  console.log(i) // 0 1 3 4
}
```

## 六、四种for适用场景对比
| 循环         | 适用场景                 | 优缺点                           |
| ------------ | ------------------------ | -------------------------------- |
| for          | 精准计数、倒序、复杂步长 | 灵活，代码偏长                   |
| for in       | 纯对象键名遍历           | 不适合数组，遍历原型属性         |
| for of       | 数组、字符串、Map、Set   | 简洁，拿值方便，不能直接遍历对象 |
| for await of | 异步迭代                 | 处理Promise流式数据              |

## 七、常见坑
1. for 用 var 变量泄露
```js
// 错误
for(var i=0;i<3;i++){}
console.log(i) // 3

// 正确
for(let i=0;i<3;i++){}
console.log(i) // 报错 i未定义
```
2. for in 遍历数组得到字符串下标
3. for of 不能遍历普通对象，对象用 `Object.keys` 搭配 for of：
```js
const obj = {a:1,b:2}
for(let k of Object.keys(obj)){
  console.log(k, obj[k])
}
```