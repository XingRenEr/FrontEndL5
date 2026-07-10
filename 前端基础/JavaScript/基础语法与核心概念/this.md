# JS this 全方位详解
## 核心口诀
**`this` 不看定义在哪，只看调用方式**，一共 5 种绑定规则，优先级从高到低：
`new绑定 > 显式绑定(call/apply/bind) > 隐式绑定(对象.方法) > 默认绑定(全局/严格模式) > 箭头函数(不绑定this，继承外层)`

## 1. 默认绑定：普通函数直接调用
### 非严格模式
普通函数单独执行，`this` 指向**全局对象**
- 浏览器：`window`
- Node 脚本：`globalThis`
```js
function fn() {
  console.log(this); // window
  this.a = 10;
}
fn();
console.log(a); // 10
```

### 严格模式 `"use strict"`
普通函数直接调用，`this = undefined`，赋值直接报错
```js
"use strict";
function fn() {
  console.log(this); // undefined
  this.a = 10; // Uncaught TypeError
}
fn();
```

## 2. 隐式绑定：对象.方法调用
函数作为对象的属性调用，`this` = **调用时前面的对象**
```js
const obj = {
  name: "张三",
  say: function () {
    console.log(this.name);
  }
};
obj.say(); // 张三，this 是 obj
```
### 坑：别名赋值丢失绑定
```js
let fn = obj.say;
fn(); // 默认绑定，window / undefined(严格模式)
```

### 链式调用，只看最后一层
```js
const obj1 = {
  fn: function () { console.log(this) }
};
const obj2 = { o: obj1 };
obj2.o.fn(); // this = obj1
```

## 3. 显式绑定：call / apply / bind
手动指定函数内部 `this`
### call、apply（立即执行）
- `fn.call(指定this, 参数1, 参数2)`
- `fn.apply(指定this, [参数数组])`
```js
function say(age) {
  console.log(this.name, age);
}
const user = { name: "李四" };
say.call(user, 20); // 李四 20
say.apply(user, [22]); // 李四 22
```

### bind（返回新函数，永久绑定this，不立即执行）
```js
const boundFn = say.bind(user, 18);
boundFn(); // 李四 18
```
> bind 优先级高于 call/apply，bind 绑定后无法被 call 修改 this

## 4. new 绑定：构造函数实例化
使用 `new` 调用函数时，自动做4件事：
1. 创建空新对象
2. 新对象绑定为函数内 `this`
3. 执行构造函数代码（给新对象挂载属性）
4. 若无手动 return 对象，默认返回新对象
```js
function Person(name) {
  this.name = name; // this 是新建实例
}
const p = new Person("小明");
console.log(p.name); // 小明
```
坑：构造函数不加 new，退化成**默认绑定**

## 5. 箭头函数 ()=>{} 无this绑定
### 关键特性
1. **箭头函数不拥有自己的 this**，定义时**继承外层作用域的 this**
2. 不适用 new、call、apply、bind，无法修改 this
3. 不能做构造函数，new 箭头函数直接报错

```js
const obj = {
  name: "小红",
  fn: () => {
    console.log(this.name);
  }
};
obj.fn(); 
// 外层全局this，浏览器输出空（window无name）
```

### 经典场景：定时器保留this
普通函数定时器this指向window，箭头函数继承外层this
```js
const obj = {
  num: 10,
  run: function () {
    setTimeout(() => {
      console.log(this.num); // this = obj，输出10
    }, 100);
  }
};
obj.run();
```

## 6. DOM事件处理函数
DOM 事件回调里，`this` 指向**触发事件的DOM元素**
```html
<button id="btn">点击</button>
<script>
  const btn = document.getElementById("btn");
  btn.onclick = function () {
    console.log(this); // <button> DOM元素
  };
</script>
```
箭头函数写事件：this 继承外层，不再指向DOM

## 7. class 类中的 this
类内普通方法，实例调用时 this = 实例；
类方法单独提取调用会丢失this（变成undefined，类默认严格模式）
```js
class User {
  constructor() {
    this.name = "类实例";
  }
  say() {
    console.log(this.name);
  }
}
const u = new User();
u.say(); // 类实例

const fn = u.say;
fn(); // 严格模式 this=undefined，报错
```

## 绑定优先级排序（从高到低）
1. `new`
2. `bind` 硬绑定
3. `call / apply`
4. 对象隐式调用 `obj.fn()`
5. 默认绑定（普通函数直接执行）
6. 箭头函数：无视以上规则，继承外层this

## 高频面试速记
1. 普通函数直接跑 → window / undefined(严格)
2. 对象点调用 → 前面的对象
3. call/apply/bind → 第一个参数
4. new 函数 → 新建实例
5. 箭头函数 → 定义位置外层this，永不改变
6. DOM事件回调 → 当前DOM元素