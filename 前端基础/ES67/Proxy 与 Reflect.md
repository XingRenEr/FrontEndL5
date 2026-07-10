# ES6 Proxy & Reflect 完整精讲（4大模块：Proxy方法、this问题、Reflect静态API、Proxy实现观察者模式）
## 前置基础
Proxy 用于**拦截对象的底层操作**，做代理劫持；
Reflect 提供**对象底层操作的函数式API**，完美配套 Proxy，替代 `Object`、`delete`、`in` 等命令式操作。
两者均为 ES6 标准，配合使用是响应式、数据劫持、框架底层核心原理（Vue2 Object.defineProperty 升级方案）。

# 24. Proxy 实例的拦截方法（handler 捕获器）
## 1. Proxy 语法
```js
const target = {}; // 被代理原始对象
const handler = { /* 捕获器：拦截操作 */ };
const proxy = new Proxy(target, handler); // proxy 代理实例
```
所有操作 `proxy.xxx` 都会触发 handler 对应捕获器，未定义捕获器则直接转发到 target。

## 2. 全部 handler 拦截方法（13种）
| 捕获器方法                               | 拦截操作                                                     | 参数                                  |
| ---------------------------------------- | ------------------------------------------------------------ | ------------------------------------- |
| `get(target, prop, receiver)`            | 读取属性：`proxy.xxx` / `proxy[xxx]`                         | target原始对象、属性名、代理实例proxy |
| `set(target, prop, value, receiver)`     | 赋值属性：`proxy.xxx = 1`                                    | target、属性、新值、proxy             |
| `has(target, prop)`                      | `xxx in proxy` 判断属性存在                                  | target、属性                          |
| `deleteProperty(target, prop)`           | `delete proxy.xxx` 删除属性                                  | target、属性                          |
| `apply(target, thisArg, args)`           | 函数调用 `proxy()`、`proxy.call()`                           | target函数、this指向、参数数组        |
| `construct(target, args, newTarget)`     | `new proxy()` 实例化                                         | target构造函数、参数、new出来的实例   |
| `getPrototypeOf(target)`                 | `Object.getPrototypeOf(proxy)`                               | target                                |
| `setPrototypeOf(target, proto)`          | `Object.setPrototypeOf(proxy, proto)`                        | target、新原型                        |
| `isExtensible(target)`                   | `Object.isExtensible(proxy)`                                 | target                                |
| `preventExtensions(target)`              | `Object.preventExtensions(proxy)`                            | target                                |
| `getOwnPropertyDescriptor(target, prop)` | `Object.getOwnPropertyDescriptor(proxy, prop)`               | target、属性                          |
| `defineProperty(target, prop, desc)`     | `Object.defineProperty(proxy, prop, desc)`                   | target、属性、描述符                  |
| `ownKeys(target)`                        | `Object.keys(proxy)` / `Object.getOwnPropertyNames` / `Reflect.ownKeys` | target                                |

## 3. 常用捕获器示例
### get / set（最常用，数据监听）
```js
const user = { name: "张三" };
const proxy = new Proxy(user, {
  get(target, key) {
    console.log(`读取属性：${key}`);
    return target[key];
  },
  set(target, key, val) {
    console.log(`修改属性：${key}=${val}`);
    target[key] = val;
    return true; // set必须返回布尔，代表赋值成功
  }
});

proxy.name; // 读取属性：name
proxy.age = 18; // 修改属性：age=18
```

### has 拦截 in 判断
```js
const p = new Proxy({a:1}, {
  has(target, key) {
    console.log(`判断是否存在${key}`);
    return Reflect.has(target, key);
  }
})
console.log("a" in p);
```

### apply 拦截函数调用
```js
function sum(a,b) { return a+b }
const pSum = new Proxy(sum, {
  apply(target, ctx, args) {
    console.log("函数被调用", args);
    return Reflect.apply(target, ctx, args);
  }
})
pSum(1,2);
```

### construct 拦截 new
```js
class Person {}
const P = new Proxy(Person, {
  construct(target, args) {
    console.log("new 实例化");
    return new target(...args);
  }
})
new P();
```

## 4. 限制：不可代理的场景
1. 原始值（数字、字符串、布尔）不能直接代理，必须包对象；
2. 部分内置对象（Map/Set/Date）部分底层操作无法完整拦截；
3. target 不可扩展、属性不可配置时，对应捕获器会报错。

# 25. Proxy 中的 this 指向问题（高频坑点）
## 核心结论
**当通过 proxy 访问 target 内部方法时，方法内的 `this` 指向 proxy，而非原始 target**。
底层原理：捕获器第三个参数 `receiver` 永远是 proxy 实例，对象方法读取时 receiver 作为 this 传入。

## 案例1：普通对象 this 错乱
```js
const target = {
  name: "原始对象",
  fn() {
    console.log(this.name);
  }
};
const proxy = new Proxy(target, {});

target.fn(); // 原始对象（this=target）
proxy.fn();  // undefined（this=proxy，proxy无name属性）
```

## 案例2：私有属性 # 完全失效（经典坑）
私有字段 `#xxx` 只能被原始对象实例访问，this 变成 proxy 后直接报错：
```js
const target = {
  #secret: 123,
  getSecret() {
    return this.#secret;
  }
};
const proxy = new Proxy(target, {});
proxy.getSecret(); // Uncaught TypeError: Cannot read private member
```

## 案例3：Date、Map 内置对象 this 异常
内置类依赖内部插槽，this 变为 proxy 会丢失内部状态：
```js
const map = new Map();
const pMap = new Proxy(map, {});
pMap.set("a", 1); // 报错，Map内部方法this必须是真实Map实例
```

## 解决方案
### 方案1：Reflect 绑定 target 为 this（推荐）
在 get 捕获器中判断函数，使用 `bind(target)` 修正 this：
```js
const target = {
  name: "原始对象",
  fn() { console.log(this.name) }
};
const proxy = new Proxy(target, {
  get(target, key, receiver) {
    const res = Reflect.get(target, key, receiver);
    // 如果取到的是函数，绑定this为原始target
    if (typeof res === "function") {
      return res.bind(target);
    }
    return res;
  }
});
proxy.fn(); // 原始对象，修复完成
```

### 方案2：拆分代理，单独代理数据，不代理内置实例
复杂内置对象不要直接代理，只代理普通数据层。

# 26. Reflect 静态方法（配套 Proxy 的底层API）
## 1. 设计目的
1. 将对象底层操作转为**函数调用**，替代命令式操作 `delete`/`in`/`new`；
2. 与 Proxy 13个捕获器一一对应，每个捕获器都有同名 Reflect 方法；
3. 统一返回值规范（布尔值表示操作成功），避免抛错；
4. 完美处理 receiver，解决 Proxy this 传递问题。

## 2. Reflect 全部静态方法（13个，和Proxy捕获器一一对应）
| Reflect 方法                               | 等价原生操作                      | 返回值           |
| ------------------------------------------ | --------------------------------- | ---------------- |
| `Reflect.get(target, prop, receiver)`      | `target[prop]`                    | 属性值           |
| `Reflect.set(target, prop, val, receiver)` | `target[prop] = val`              | boolean          |
| `Reflect.has(target, prop)`                | `prop in target`                  | boolean          |
| `Reflect.deleteProperty(target, prop)`     | `delete target[prop]`             | boolean          |
| `Reflect.apply(fn, thisArg, args)`         | `fn.call(thisArg, ...args)`       | 函数返回值       |
| `Reflect.construct(Fn, args, newTarget)`   | `new Fn(...args)`                 | 新实例           |
| `Reflect.getPrototypeOf(target)`           | `Object.getPrototypeOf`           | 原型对象         |
| `Reflect.setPrototypeOf(target, proto)`    | `Object.setPrototypeOf`           | boolean          |
| `Reflect.isExtensible(target)`             | `Object.isExtensible`             | boolean          |
| `Reflect.preventExtensions(target)`        | `Object.preventExtensions`        | boolean          |
| `Reflect.getOwnPropertyDescriptor(t, p)`   | `Object.getOwnPropertyDescriptor` | 描述符/undefined |
| `Reflect.defineProperty(t, p, desc)`       | `Object.defineProperty`           | boolean          |
| `Reflect.ownKeys(target)`                  | `Object.keys + Symbol属性`        | 所有属性数组     |

## 3. Reflect 对比原生 Object API 的优势
1. **返回布尔标识操作结果**
`Object.defineProperty` 失败直接抛错；`Reflect.defineProperty` 返回 false，不报错：
```js
const obj = {};
Object.freeze(obj);
Reflect.defineProperty(obj, "a", {value:1}); // false
// Object.defineProperty(obj, "a", {value:1}) // 直接报错
```

2. **支持 receiver，精准控制 this**
`Reflect.get(target, key, receiver)` 读取时，对象getter内this指向receiver，是Proxy核心配套。

3. **无全局污染，纯静态方法**
不像 `delete`、`in` 是操作符，统一函数式调用，便于封装。

## 4. Proxy + Reflect 标准写法（最佳实践）
handler 捕获器内部优先用 Reflect 转发操作，保证原有逻辑完整：
```js
const proxy = new Proxy(obj, {
  get(target, key, receiver) {
    console.log("读取", key);
    // 转发原始读取逻辑，保留receiver上下文
    return Reflect.get(target, key, receiver);
  },
  set(target, key, val, receiver) {
    console.log("修改", key, val);
    // 转发赋值，返回布尔给Proxy校验
    return Reflect.set(target, key, val, receiver);
  }
})
```

# 27. 实战：Proxy + Reflect 实现观察者模式（简易响应式）
## 需求说明
1. 定义依赖收集容器：存储当前正在执行的渲染函数；
2. 通过 Proxy 劫持对象读写：
   - 读取属性时：收集当前渲染函数（依赖收集）
   - 修改属性时：执行收集到的所有渲染函数（派发更新）
3. 采用 Reflect 完成底层转发，规避 this 问题。

## 完整可运行代码
```js
// 1. 全局保存当前正在执行的副作用函数（渲染/更新函数）
let activeEffect = null;

// 2. 依赖存储：key -> 副作用函数集合
const targetMap = new WeakMap();

// 注册副作用（页面更新逻辑）
function watchEffect(fn) {
  activeEffect = fn;
  fn(); // 立即执行一次，触发get收集依赖
  activeEffect = null;
}

// 收集依赖：读取属性时调用
function track(target, key) {
  if (!activeEffect) return;
  // 拿到对象对应的依赖Map
  let depsMap = targetMap.get(target);
  if (!depsMap) targetMap.set(target, (depsMap = new Map()));
  // 拿到属性对应的副作用集合
  let deps = depsMap.get(key);
  if (!deps) depsMap.set(key, (deps = new Set()));
  deps.add(activeEffect);
}

// 派发更新：修改属性时执行所有副作用
function trigger(target, key) {
  const depsMap = targetMap.get(target);
  if (!depsMap) return;
  const deps = depsMap.get(key);
  deps && deps.forEach(fn => fn());
}

// 3. 响应式代理工厂（Proxy+Reflect封装）
function reactive(target) {
  return new Proxy(target, {
    get(target, key, receiver) {
      const res = Reflect.get(target, key, receiver);
      track(target, key); // 读取收集依赖
      return res;
    },
    set(target, key, value, receiver) {
      const oldVal = Reflect.get(target, key, receiver);
      const success = Reflect.set(target, key, value, receiver);
      if (oldVal !== value) {
        trigger(target, key); // 值变化触发更新
      }
      return success;
    }
  })
}

// ========== 使用示例 ==========
const state = reactive({
  count: 0,
  name: "小明"
});

// 注册更新函数
watchEffect(() => {
  console.log("页面渲染：", state.count, state.name);
});

state.count = 10; // 触发更新，打印页面渲染
state.name = "小红"; // 触发更新，打印页面渲染
```

## 代码核心逻辑拆解
1. **track 依赖收集**
当 `watchEffect` 执行渲染函数，读取 `state.count` 触发 proxy get，把当前渲染函数存入对应属性的集合；
2. **trigger 派发更新**
修改 `state.count` 触发 proxy set，取出该属性所有收集的函数批量执行；
3. **Reflect 作用**
- `Reflect.get/set` 保留原始对象读写逻辑；
- 传递 receiver 保证 getter 内 this 指向正确；
- set 返回布尔值满足 Proxy 捕获器规范。

## 拓展优化点（进阶）
1. 分支清理：避免重复收集无效依赖；
2. 调度器：控制更新执行顺序（异步批量更新）；
3. 深层对象递归代理：实现深度响应；
4. 数组劫持：拦截 push、splice 等数组方法。

# 整体总结
1. **Proxy**：拦截对象13种底层操作，实现数据劫持，核心捕获器 get/set/apply/construct；
2. **this 坑**：代理对象方法内this指向proxy，私有属性、内置对象会报错，通过bind或Reflect修正；
3. **Reflect**：与Proxy捕获器一一对应的静态API，函数式操作底层对象，统一返回值，配套Proxy标准写法；
4. **观察者模式**：Proxy劫持读写 + Reflect转发底层逻辑 + track/trigger 依赖收集更新，是Vue3响应式核心原理。