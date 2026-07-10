# ES6 Set / WeakSet / Map / WeakMap 完整详解
## 一、Set（ES6）
### 1. 基础概念
`Set` 是**类数组集合**，特点：
1. 成员**唯一不重复**，自动去重；
2. 无键值对，只有值；
3. 可存储任意类型：基本类型、引用类型；
4. 有序：插入顺序遍历；
5. **强引用**，不参与垃圾回收优化。

### 2. 创建与初始化
```js
// 空集合
const s1 = new Set();
// 数组初始化（自动去重）
const s2 = new Set([1,2,2,3,3]);
console.log(s2); // Set(3) {1,2,3}
```

### 3. 常用实例属性 & 方法
#### 属性
- `size`：返回成员数量（只读）
```js
s2.size // 3
```

#### 操作方法（增删查）
1. `add(value)`：添加值，返回自身，可链式调用
```js
s1.add(1).add(2).add(1); // 重复添加无效
```
2. `has(value)`：判断是否存在，返回布尔
```js
s1.has(1); // true
s1.has(99); // false
```
3. `delete(value)`：删除指定值，返回布尔（是否删除成功）
```js
s1.delete(2); // true
```
4. `clear()`：清空所有成员，无返回值
```js
s1.clear();
```

#### 遍历方法（4种）
Set 没有下标，不能用 `for` 下标循环，只能遍历：
1. `keys()` / `values()`：Set 键和值相同，两者返回一样迭代器
2. `entries()`：返回 `[value, value]` 键值对迭代器
3. `forEach()`：遍历每个成员
4. `for...of`：最常用

```js
const set = new Set(['a','b']);
// for of
for (const v of set) console.log(v);
// forEach
set.forEach((val) => console.log(val));
```

### 4. Set 实用场景
1. 数组去重
```js
const arr = [1,2,2,3];
const unique = [...new Set(arr)]; // [1,2,3]
```
2. 交集、并集、差集
```js
const a = new Set([1,2,3]);
const b = new Set([2,3,4]);
// 并集
const union = new Set([...a, ...b]);
// 交集
const intersect = new Set([...a].filter(x => b.has(x)));
// a差集b
const diff = new Set([...a].filter(x => !b.has(x)));
```

### 5. 缺陷
无法直接通过下标取值，没有 `get` 方法；存储对象时是强引用，无法自动释放内存。

---

## 二、WeakSet（ES6）
### 1. 核心特性（和 Set 最大区别）
1. **只能存储引用类型**（对象、数组、函数），不能存数字/字符串等原始值；
2. **弱引用**：内部对对象是弱引用，若外部无其他变量引用该对象，GC（垃圾回收）会自动回收，WeakSet 自动清除该成员；
3. **不可遍历**：没有 `size`、`keys/values/entries/forEach`，不能 `for...of`；
4. 只有 3 个方法：`add / has / delete`，无 `clear`。

### 2. 示例
```js
const ws = new WeakSet();
let obj = { name: "test" };
ws.add(obj);
ws.has(obj); // true

obj = null; // 取消外部引用
// 引擎GC触发后，WeakSet自动移除obj，无法手动感知
```

### 3. 使用场景
临时存放 DOM 对象、临时标记对象，不阻碍垃圾回收，防止内存泄漏。
> 例：标记页面已渲染的DOM节点，节点销毁后自动释放。

### 4. 限制总结
- 不能存基本数据类型
- 不能遍历、无法获取长度size
- 成员自动被GC清理，无法预知内部数据

---

## 三、Map（ES6）
### 1. 基础概念
`Object` 的升级版键值对结构，解决对象键只能是字符串/Symbol 的痛点：
1. **键可以是任意类型**：数字、对象、数组、函数、Symbol 都能当 key；
2. 键唯一，重复赋值会覆盖；
3. 有序：按插入顺序存储；
4. 强引用；
5. 拥有 `size` 获取键值对数量。

### 2. 创建初始化
```js
// 空Map
const m1 = new Map();
// 二维数组初始化
const m2 = new Map([
  ['name', '张三'],
  [100, '分数'],
  [{id:1}, '对象键']
]);
```

### 3. 属性与核心方法
#### 属性
- `size`：键值对总数

#### 增删查改
1. `set(key, value)`：设置键值，重复key覆盖，返回自身可链式
```js
m1.set('age', 18).set('age', 20); // age最终=20
```
2. `get(key)`：根据key取值，不存在返回 `undefined`
```js
m1.get('age'); // 20
```
3. `has(key)`：判断key是否存在
4. `delete(key)`：删除指定键，返回布尔
5. `clear()`：清空全部键值对

#### 遍历（全部可遍历）
1. `keys()`：所有键迭代器
2. `values()`：所有值迭代器
3. `entries()`：`[key, value]` 迭代器（Map 默认迭代器）
4. `forEach((val, key, map) => {})`
5. `for...of`

```js
const map = new Map([['a',1],['b',2]]);
for (const [k, v] of map) {
  console.log(k, v);
}
```

### 4. Map vs Object 对比
| 特性   | Object                      | Map                      |
| ------ | --------------------------- | ------------------------ |
| 键类型 | String / Symbol             | 任意类型（对象、数字等） |
| 顺序   | ES6后有序，但原型自带键干扰 | 严格插入顺序，无内置键   |
| 长度   | Object.keys(obj).length     | 直接 .size               |
| 遍历   | 需手动拿key遍历             | 原生迭代器，直接for...of |
| 性能   | 少量数据友好                | 频繁增删大量数据性能更好 |

### 5. 使用场景
需要复杂类型作为键、大量键值对频繁增删、需要精准获取长度。

---

## 四、WeakMap（ES6）
### 1. 核心特性（和 Map 区别）
1. **key 只能是引用类型（对象/数组）**，value 无限制；原始值不能做键；
2. **键是弱引用**：若作为key的对象外部无其他引用，GC自动回收，WeakMap 自动移除这条键值对；
3. **不可遍历**：无 `size`、无 `keys/values/entries/forEach`，不能 `for...of`；
4. 只有4个方法：`set / get / has / delete`，无 `clear`。

### 2. 代码示例
```js
const wm = new WeakMap();
let user = { id: 1 };
wm.set(user, '用户数据');
wm.get(user); // 用户数据

user = null; // 外部引用销毁
// GC触发后，{id:1} 对象被回收，wm中这条记录自动消失
```

### 3. 典型应用场景（解决内存泄漏）
1. DOM 节点缓存数据
给DOM绑定附加数据，DOM销毁后自动清除缓存，不占用内存
```js
const domData = new WeakMap();
const div = document.querySelector('div');
domData.set(div, { scrollTop: 0 });
// div 被移除销毁，自动释放缓存
```
2. 对象私有数据存储
存放实例私有属性，实例销毁自动释放，不污染全局。

### 4. 限制
- key 只能对象，不支持基础类型
- 不能遍历、无法获取总数size
- 内部成员随GC自动销毁，无法枚举全部数据

---

# 四组结构核心对比总结
| 结构    | 存储内容             | 引用类型 | 可遍历 | size | 核心特点             |
| ------- | -------------------- | -------- | ------ | ---- | -------------------- |
| Set     | 任意类型值           | 强引用   | ✅      | ✅    | 元素去重，无键       |
| WeakSet | 仅对象               | 弱引用   | ❌      | ❌    | 自动GC，标记对象     |
| Map     | 任意key+任意value    | 强引用   | ✅      | ✅    | key不限类型，键值对  |
| WeakMap | key仅对象，value任意 | 键弱引用 | ❌      | ❌    | 对象销毁自动清除缓存 |

# 关键记忆点
1. **Weak 系列共性**：弱引用、不可遍历、无size、只能存对象；作用是**自动垃圾回收，防止内存泄漏**
2. Set 存值去重，Map 存键值对；
3. 普通 Set/Map 强引用，对象不手动清空就不会被GC；
4. WeakSet 存对象本身，WeakMap 用对象当key存附加数据。