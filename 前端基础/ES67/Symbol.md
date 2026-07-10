# ES6 Symbol 完整详解（含 Symbol.for、Symbol.keyFor、内置Symbol）
## 一、Symbol 基础回顾
ES6 引入第七种原始数据类型：`Symbol`，用于生成**唯一、不可重复**的值，解决对象属性名冲突问题。
1. 创建方式：`const s = Symbol('描述文本')`
2. 特点：
   - 每次调用 `Symbol()` 都会生成全新唯一值，哪怕描述相同
   - 不能用 `new Symbol()`，会报错（原始类型，不是构造函数）
   - 不能隐式转数字，可显式转字符串/布尔
   - 作为对象键时，无法被 `for...in`、`Object.keys()` 遍历，需用 `Object.getOwnPropertySymbols()` 获取

```js
const s1 = Symbol('foo')
const s2 = Symbol('foo')
console.log(s1 === s2) // false，普通Symbol互不相等
```

---

# 22. Symbol.for() & Symbol.keyFor() 全局注册表
## 1. Symbol.for(key) 全局复用Symbol
普通 `Symbol()` 是**局部独立**，`Symbol.for()` 会维护一张**全局Symbol注册表**：
1. 接收字符串参数 `key`
2. 先去全局注册表查找是否存在该key对应的Symbol
   - 存在：直接返回已有的Symbol
   - 不存在：新建Symbol存入全局注册表，再返回
3. 核心：**相同key一定返回同一个Symbol实例**

```js
// 第一次，全局无foo，新建存入注册表
const sA = Symbol.for('foo')
// 第二次，全局已有foo，直接取出同一个Symbol
const sB = Symbol.for('foo')
console.log(sA === sB) // true

// 对比普通Symbol
const sC = Symbol('foo')
console.log(sA === sC) // false，普通Symbol不进全局表
```

### 使用场景
跨模块、跨文件共享同一个Symbol常量，保证全局唯一标识。

## 2. Symbol.keyFor(symbol) 获取全局Symbol的key
作用：**只能查询全局注册表内**由 `Symbol.for()` 创建的Symbol，返回它注册时的字符串key；
如果传入普通 `Symbol()` 创建的值，返回 `undefined`。

```js
const s1 = Symbol.for('user')
console.log(Symbol.keyFor(s1)) // "user"

const s2 = Symbol('user')
console.log(Symbol.keyFor(s2)) // undefined，未存入全局注册表
```

### 完整示例结合两者
```js
// 注册全局Symbol
const token = Symbol.for('token-key')
// 反向取出注册名
const key = Symbol.keyFor(token)
console.log(key) // token-key

// 再次通过key拿到同一个Symbol
const token2 = Symbol.for(key)
console.log(token === token2) // true
```

### 关键区分表
| 创建方式          | 是否存入全局注册表 | Symbol.keyFor 查询结果 | 相同描述是否相等 |
| ----------------- | ------------------ | ---------------------- | ---------------- |
| Symbol('xxx')     | ❌ 不存入           | undefined              | false            |
| Symbol.for('xxx') | ✅ 存入             | 返回对应字符串key      | true             |

---

# 23. ES6 内置 Symbol 值（系统内置Symbol常量）
JS 内置 11 个 Symbol 常量，统称为**Well-known Symbols**，用于**自定义语言底层行为**：
比如改变对象的迭代、类型转换、实例检测、字符串匹配等原生逻辑，所有内置Symbol都不能用 `Symbol.keyFor()` 获取（不在全局注册表）。

## 完整内置Symbol清单 + 用法讲解
### 1. Symbol.hasInstance
对应 `instanceof` 运算符底层逻辑
当执行 `obj instanceof Class`，实际调用 `Class[Symbol.hasInstance](obj)`，可重写自定义判断规则。
```js
class CheckNum {
  static [Symbol.hasInstance](target) {
    return typeof target === 'number'
  }
}
console.log(123 instanceof CheckNum) // true
console.log('abc' instanceof CheckNum) // false
```

### 2. Symbol.isConcatSpreadable
控制数组/类数组调用 `concat()` 时是否展开
- `true`：concat 时展开元素
- `false`：整体作为单个元素拼接
```js
const arr = [1,2]
arr[Symbol.isConcatSpreadable] = false
console.log([0].concat(arr)) // [0, [1,2]] 不展开
```

### 3. Symbol.species
衍生对象创建时使用的构造函数（Array、Promise、Map 等内置类用到）
例如数组调用 `map/filter` 生成新数组时，会读取 `this.constructor[Symbol.species]` 作为新数组构造器。

### 4. Symbol.match
字符串 `str.match(obj)` 底层调用 `obj[Symbol.match](str)`，自定义匹配逻辑，正则原型自带该Symbol。

### 5. Symbol.replace
`str.replace(obj, fn)` 底层触发 `obj[Symbol.replace](str, fn)`，自定义替换规则。

### 6. Symbol.search
`str.search(obj)` 底层调用 `obj[Symbol.search](str)`，自定义查找索引逻辑。

### 7. Symbol.split
`str.split(obj)` 底层调用 `obj[Symbol.split](str)`，自定义分割逻辑。

### 8. Symbol.iterator
对象默认迭代器，`for...of`、扩展运算符 `...`、`Array.from` 都会读取该方法，返回迭代器对象。
**最常用内置Symbol**
```js
const obj = {
  data: [10,20,30],
  [Symbol.iterator]() {
    let index = 0
    const arr = this.data
    return {
      next() {
        return index < arr.length 
          ? {value: arr[index++], done: false} 
          : {done: true}
      }
    }
  }
}
// 可for...of遍历
for(let v of obj) console.log(v) // 10 20 30
```

### 9. Symbol.toPrimitive
对象转原始值时的底层方法，接收参数 `hint`（number/string/default），自定义对象加减、拼接时的转换结果。
```js
const user = {
  age: 20,
  [Symbol.toPrimitive](hint) {
    if(hint === 'number') return this.age
    if(hint === 'string') return `年龄${this.age}`
    return this.age
  }
}
console.log(+user) // 20
console.log(`${user}`) // 年龄20
```

### 10. Symbol.toStringTag
`Object.prototype.toString.call(obj)` 返回的类型标识，自定义对象输出的 `[object Xxx]` 文本。
```js
const demo = {
  [Symbol.toStringTag]: "MyCustomObj"
}
console.log(Object.prototype.toString.call(demo)) 
// "[object MyCustomObj]"
```

### 11. Symbol.unscopables
控制 `with` 语法环境下哪些属性不会被绑定到with作用域，现代开发极少使用。

## 内置Symbol通用特性
1. 全部是只读常量，挂载在 `Symbol` 对象上
2. 不属于全局注册表，`Symbol.keyFor(Symbol.iterator)` 返回 `undefined`
3. 作用是**拦截、重写JS原生语法行为**，实现元编程
4. 常用来实现自定义迭代、自定义正则匹配、自定义类型转换等高级封装

---

# 综合面试核心考点总结
1. `Symbol()` vs `Symbol.for()`
   - Symbol：局部唯一，不进全局表，keyFor返回undefined
   - Symbol.for：全局单例，存入注册表，同key复用
2. Symbol.keyFor 只能识别 Symbol.for 创建的全局Symbol
3. 内置Symbol（Well-known Symbol）用于元编程，改写语言底层操作
4. 高频内置Symbol：`Symbol.iterator`（迭代）、`Symbol.toStringTag`（类型判断）、`Symbol.toPrimitive`（类型转换）、`Symbol.hasInstance`（instanceof）