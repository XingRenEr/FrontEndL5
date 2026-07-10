# JS JSON 完整讲解
## 一、JSON 是什么
1. JSON 全称：**JavaScript Object Notation**，JS 对象标记语言
2. 一种**轻量级数据交换格式**，前后端传输数据标准
3. 本质是**字符串**，和 JS 对象语法相似，但有严格语法限制
4. 独立语言格式，Java/Python/Go 都能解析，不局限 JS

## 二、JSON 严格语法规则（和 JS 对象区别重点）
1. **键名必须用双引号 `"key"`**，单引号/无引号全部非法
2. 值只能是这 6 种类型：
   - 字符串（双引号）、数字、布尔 `true/false`、`null`、数组、对象
3. **不能有**：`undefined`、函数、Date、正则、注释 `// /* */`
4. 末尾不能加逗号（数组/对象最后一项后禁止 `,`）
5. 不能使用单引号包裹字符串

合法 JSON 示例（字符串）：
```json
{
  "name": "小明",
  "age": 18,
  "isStudent": true,
  "hobby": ["篮球", "看书"],
  "info": {
    "address": null
  }
}
```

非法示例：
```js
// 1. 键无引号
{ name: "小明" }
// 2. 单引号
{ 'name': "小明" }
// 3. 末尾逗号
{"a":1, "b":2,}
// 4. 函数、undefined
{"fn": ()=>{}, "data": undefined}
```

## 三、JS 两大核心方法（JSON 转换）
### 1. JSON.stringify()  JS对象 → JSON字符串
作用：将 JS 对象/数组转为符合规范的 JSON 字符串
#### 基础用法
```js
const obj = { name: "小红", age: 20 };
const jsonStr = JSON.stringify(obj);
console.log(jsonStr);
// {"name":"小红","age":20}
```

#### 三个参数完整语法
`JSON.stringify(value, replacer, space)`
1. `value`：要转换的数据
2. `replacer` 过滤器：数组/函数，筛选保留字段
```js
// 只序列化 name、age 两个key
JSON.stringify(obj, ["name", "age"])

// 函数过滤，修改值
JSON.stringify(obj, (key, val) => {
  if(key === "age") return val + 1;
  return val;
})
```
3. `space` 格式化缩进（数字/制表符）
```js
// 缩进2空格，格式化输出，方便阅读
JSON.stringify(obj, null, 2)
```

#### stringify 自动忽略的值
转换时会直接丢弃：`undefined`、函数、Symbol
- 对象内：忽略该键
- 数组内：转为 `null`
```js
const test = { a: undefined, b: ()=>{}, c: Symbol() };
JSON.stringify(test); // "{}"
```

#### 自定义序列化：toJSON
对象自定义 `toJSON` 方法，stringify 会优先读取它的返回值
```js
const user = {
  name: "小李",
  toJSON() {
    return { nickname: this.name }
  }
};
JSON.stringify(user); // {"nickname":"小李"}
```

### 2. JSON.parse() JSON字符串 → JS对象
作用：解析 JSON 字符串，还原成 JS 对象/数组
```js
const str = '{"name":"张三","age":22}';
const obj = JSON.parse(str);
console.log(obj.name); // 张三
```

#### 第二个参数：reviver 转换回调
遍历每一对键值，可对数据做格式化（如时间字符串转Date）
```js
const s = '{"time":"2026-07-10","num":"100"}';
const res = JSON.parse(s, (key, val) => {
  if(key === "num") return Number(val);
  return val;
});
```

#### 报错场景
字符串不符合 JSON 规范时抛出 `SyntaxError`，建议包 try/catch
```js
function safeParse(str) {
  try {
    return JSON.parse(str);
  } catch (err) {
    console.error("JSON解析失败", err);
    return null; // 出错返回默认值
  }
}
```

## 四、JS对象 vs JSON 对照表
| 对比项   | JS 对象                             | JSON 字符串                       |
| -------- | ----------------------------------- | --------------------------------- |
| 本质     | 内存中的引用数据类型                | 纯文本字符串                      |
| 键名     | 双引号、单引号、无引号都可以        | 必须双引号                        |
| 字符串   | 单/双引号都行                       | 只能双引号                        |
| 支持值   | 函数、undefined、Date、正则、Symbol | 仅数字/布尔/null/数组/对象/字符串 |
| 末尾逗号 | 允许                                | 禁止                              |
| 注释     | 支持 // /* */                       | 完全不支持                        |

## 五、常见实用场景
1. **本地存储 localStorage / sessionStorage**
Storage 只能存字符串，对象必须序列化
```js
// 存
localStorage.setItem("user", JSON.stringify({name:"小明"}));
// 取
const user = JSON.parse(localStorage.getItem("user"));
```
2. **接口请求传参**
POST 请求 body 常转 JSON 字符串发送
3. **深拷贝简单对象**
利用序列化+解析实现简易深拷贝（有缺陷）
```js
const origin = { a: 1, list: [1,2] };
const copy = JSON.parse(JSON.stringify(origin));
```
### 深拷贝缺陷（慎用）
无法拷贝：函数、undefined、Symbol、Date、正则、循环引用对象、Map/Set

## 六、高频坑
1. JSON.parse 传入非标准 JSON 直接报错，一定要捕获异常
2. 含 Date 对象经过 stringify 会变成字符串，解析后不会自动变回 Date，需要 reviver 处理
3. 循环引用对象调用 JSON.stringify 直接报错
```js
let a = {};
a.self = a;
JSON.stringify(a); // 报错
```
4. 数字极大/极小会丢失精度（超长id建议后端返回字符串）
5. 注释不能写在 JSON 里，后端带注释的字符串解析直接失败