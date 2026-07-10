# ES6/ES7 Promise 核心考点梳理
## 34. Promise 对象 — 基础与应用
### 一、Promise 是什么
ES6 新增异步解决方案，**专门处理异步操作**（接口请求、文件读写、定时器等），解决回调地狱问题。
1. 三种状态（状态不可逆）
    - `pending`：进行中（初始）
    - `fulfilled`：已成功（调用 resolve）
    - `rejected`：已失败（调用 reject）
2. 基础语法
```js
const p = new Promise((resolve, reject) => {
  // 异步逻辑
  setTimeout(() => {
    resolve('成功数据') // 触发 .then
    // reject('错误信息') // 触发 .catch
  }, 1000)
})
```

### 二、实例方法（应用核心）
1. `.then(res => {})`
接收成功结果，**返回新 Promise**，支持链式调用
```js
p.then(res => {
  console.log(res)
  return res + ' 二次处理'
}).then(res2 => console.log(res2))
```
2. `.catch(err => {})`
捕获失败/异常，可放在链式末尾统一处理错误
3. `.finally(() => {})`
无论成功失败都会执行，无参数，常用于加载结束、关闭 loading

### 三、静态方法（实际开发高频应用）
1. `Promise.all([p1,p2,p3])`
所有 Promise 全部成功才返回结果数组；**任意一个失败直接 reject**
适用：多个互不依赖接口并行请求
2. `Promise.race([p1,p2,p3])`
**谁先完成就返回谁**（无论成功失败）
适用：接口超时拦截（请求 + 延时 Promise 竞赛）
3. `Promise.allSettled([p1,p2])`
全部执行完毕再返回，每个项包含 `status+value/reason`
适用：批量请求，不在乎个别失败，需要全部结果
4. `Promise.resolve()` / `Promise.reject()`
快速创建已成功/失败的 Promise，简化代码

### 四、典型应用场景
1. 消除回调地狱，链式异步流程
2. 多个异步请求并行处理
3. 接口超时、统一错误捕获
4. 配合 ES7 `async/await` 同步写法封装异步

## 35. Promise 与宏任务、微任务（事件循环）
### 一、任务分类
1. **宏任务 macro-task**
整体代码 script、`setTimeout`、`setInterval`、AJAX、DOM 渲染、IO 操作
2. **微任务 micro-task**
`.then` / `.catch` / `.finally`、`Promise.then`、`queueMicrotask`、`async await` 内部等待逻辑

### 二、事件循环执行顺序（核心规则）
1. 先执行**同步代码**（主线程立即执行）
2. 同步代码执行完毕，清空**所有微任务队列**（一次性全部执行完）
3. 取出**一个宏任务**执行
4. 重复 2、3 循环

### 三、Promise 任务特性
- `new Promise((resolve)=>{})` 内部回调函数是**同步代码**，创建时立刻执行
- `resolve()` 仅修改状态，`.then` 回调属于**微任务**，不会立刻执行，等同步代码走完再执行

### 四、经典执行顺序示例
```js
console.log('同步1')
setTimeout(() => {
  console.log('宏任务')
}, 0)
Promise.resolve().then(() => {
  console.log('微任务Promise')
})
console.log('同步2')
```
输出顺序：
1. 同步1
2. 同步2
3. 微任务Promise
4. 宏任务

### 五、高频考点总结
1. Promise 构造器内部代码同步，`.then` 是微任务
2. 微任务优先级 > 宏任务
3. 每执行完一个宏任务，必先清空全部微任务再执行下一个宏任务
4. async/await 本质是 Promise 语法糖，await 后代码属于微任务