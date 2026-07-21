# JavaScript DOM 全套知识点整理（31~41）
## 31. DOM 方法
DOM 方法是**操作DOM对象的函数**，用于增删改查页面节点。
### 常用查询方法
```js
// 单个元素
document.getElementById("id");
document.querySelector(".class/#id/tag");
// 多个元素（集合/节点列表）
document.getElementsByClassName("cls");
document.getElementsByTagName("div");
document.querySelectorAll("div > p");
```
### 常用增删改方法
```js
// 创建节点
document.createElement("div");
document.createTextNode("文本");
// 追加/插入
parent.appendChild(child);
parent.prepend(child);
parent.insertBefore(newNode, target);
// 删除/替换
parent.removeChild(child);
child.remove();
parent.replaceChild(newNode, oldNode);
// 复制
node.cloneNode(true); // true深拷贝，false浅拷贝
```

## 32. DOM 文档（Document）
`document` 是整个HTML页面的根DOM对象，代表**整个网页文档**，是所有DOM操作入口。
1. 全局唯一对象，挂载在window上
2. 提供页面全局操作API
```js
document.body; // 获取body
document.documentElement; // 获取html根标签
document.title = "新标题";
document.write("内容"); // 页面输出文本
document.URL; // 当前页面地址
```

## 33. DOM 元素（Element）
通过查询获取到的**标签节点对象**（div/p/input等），继承Node，拥有标签专属属性。
### 核心属性
```js
let box = document.querySelector(".box");
box.innerText; // 纯文本
box.innerHTML; // 带标签文本
box.id / box.className / box.classList;
box.getAttribute("src"); // 获取自定义属性
box.setAttribute("data-id", "100");
box.removeAttribute("disabled");
```

## 34. DOM HTML
通过DOM读取/修改元素内部HTML结构，核心两个属性：
1. `innerHTML`：读写元素内完整HTML（包含标签）
```js
div.innerHTML = "<span>文字</span>";
console.log(div.innerHTML);
```
2. `innerText`：只读写纯文本，自动忽略标签，会解析换行空格
3. `textContent`：纯文本，性能更高，不解析样式

## 35. DOM CSS
JS 操作元素行内样式、类名，两种方式：
### 1. 直接操作行内样式 `.style`
```js
box.style.width = "200px";
box.style.backgroundColor = "#f00"; // 驼峰命名，不能写-
```
### 2. 通过classList操作类（推荐，分离样式与逻辑）
```js
box.classList.add("active");
box.classList.remove("hide");
box.classList.toggle("show"); // 有则删，无则加
box.classList.contains("active"); // 判断是否存在类
```
### 3. 获取计算后样式（外部CSS+行内合并）
```js
let style = window.getComputedStyle(box);
console.log(style.width);
```

## 36. DOM 导航（节点层级导航）
通过父子兄弟关系，不用选择器遍历节点：
```js
// 父级
elem.parentElement; // 父元素（只标签）
elem.parentNode; // 父节点（包含文本/注释）

// 子级
elem.children; // 子元素集合（只标签）
elem.childNodes; // 所有子节点（文本、注释、标签）

// 兄弟
elem.nextElementSibling; // 下一个兄弟元素
elem.previousElementSibling; // 上一个兄弟元素
elem.nextSibling / previousSibling // 包含空白文本节点
```

## 37. DOM 节点（Node）
DOM中所有内容都是节点，**Node是所有DOM对象的基类**，分多种类型：
1. 元素节点（标签）：nodeType = 1
2. 文本节点：nodeType = 3
3. 注释节点：nodeType = 8
4. 文档节点document：nodeType = 9
### 节点通用属性
```js
node.nodeName; // 节点名称（大写标签名/#text/#comment）
node.nodeType; // 节点类型数字
node.nodeValue; // 文本/注释内容，元素节点为null
```

## 38. DOM 事件
用户或浏览器触发的行为（点击、输入、滚动、加载等），实现页面交互。
### 常见事件分类
- 鼠标：`click`、`mouseover`、`mouseout`、`mousemove`
- 键盘：`keydown`、`keyup`、`keypress`
- 表单：`input`、`change`、`submit`、`focus`、`blur`
- 页面：`load`、`scroll`、`resize`

### 三种绑定方式
1. 行内绑定：`<div onclick="fn()"></div>`
2. DOM0级事件：`elem.onclick = function(){}`
3. DOM2级监听（推荐）：`addEventListener`

## 39. DOM 事件监听程序（addEventListener）
标准事件绑定API，支持绑定多个同类型事件、控制冒泡/捕获。
### 基础语法
```js
elem.addEventListener("click", handler, 可选参数);
function handler(e) {
  e.target; // 触发事件的元素
  e.preventDefault(); // 阻止默认行为（a跳转/表单提交）
  e.stopPropagation(); // 终止当前事件后续所有传播流程，不分捕获 / 冒泡，在哪一段调用，就截断后面所有流程
}
```
### 移除监听
```js
elem.removeEventListener("click", handler);
```
### 第三个参数
- `false`（默认）：事件冒泡
- `true`：事件捕获

## 40. DOM 集合（HTMLCollection）
动态元素集合，由 `getElementsByClassName / getElementsByTagName` 返回
1. **动态集合**：DOM变化时自动更新内容
2. 类数组，可通过下标/length遍历，**不能用数组方法forEach**
3. 只包含元素节点，无文本注释
```js
let coll = document.getElementsByTagName("div");
// 转数组才能用forEach
Array.from(coll).forEach(item=>{})
```

## 41. DOM 节点列表（NodeList）
由 `querySelectorAll / childNodes` 返回
1. **静态集合**：DOM修改不会自动更新列表内容
2. NodeList自带 `.forEach()` 遍历方法，无需转数组
3. `querySelectorAll` 的NodeList只存元素；`childNodes` 包含所有节点（文本、注释）
```js
let list = document.querySelectorAll("p");
list.forEach(p=> p.style.color="red");
```

> 更正：分两种 NodeList
> 
> 静态 NodeList（static）：querySelectorAll 返回
> 
> 获取时一次性拍摄 DOM 快照，后续 DOM 变化不会影响集合，最常用。
> 
> 动态 NodeList（live）：childNodes 返回
> 
> 和 HTMLCollection 一样实时更新，包含所有节点类型。

# HTMLCollection vs NodeList 核心区别
| 特性        | HTMLCollection   | NodeList                      |
| ----------- | ---------------- | ----------------------------- |
| 返回API     | getElementsBy*   | querySelectorAll / childNodes |
| 动态/静态   | 动态（实时更新） | 静态（快照）                  |
| 自带forEach | 无               | 有                            |
| 包含内容    | 仅元素节点       | 可包含文本、注释、元素        |

需要我把这份内容压缩成一页速记背诵版吗？
