# ES6/7 Module 模块核心考点梳理（36–41）
## 36. 模块的整体加载
### 基础语法
用 `import * as 别名` 一次性导入模块所有导出内容，形成对象统一访问。
```js
// util.js
export const a = 1;
export function fn() {}

// 主文件
import * as util from './util.js';
console.log(util.a);
util.fn();
```
### 特点
1. 整体加载是**静态导入**，编译期确定依赖，不能写在条件/循环里；
2. `*` 不会导入模块默认导出，默认导出需单独 `import xxx from`；
3. 别名不可修改，只读引用，无法直接赋值覆盖。

## 37. export 与 import 复合写法（export … from）
### 作用
中转模块，无需在当前文件接收变量再导出，常用于分层模块聚合。
### 三种常用形式
1. 导出全部
```js
export * from './util.js';
```
⚠️ 缺陷：无法导出原模块的 default 默认导出。

2. 按需导出指定变量
```js
export { a, fn } from './util.js';
```

3. 改名导出 + 导出默认
```js
// 普通导出改名
export { a as num } from './util.js';
// 单独导出默认
export { default } from './util.js';
// 默认导出改名
export { default as Util } from './util.js';
```
### 使用场景
路由聚合、工具库统一出口文件 `index.js`。

## 38. import() 动态导入（ES2020，ES7+扩展）
### 核心区别
普通 `import` 是静态编译；`import()` 是**运行时动态导入**，返回 Promise。
### 语法
```js
// 基础
import('./dialog.js').then(mod => {
  mod.show();
}).catch(err => {});

// async/await 写法
async function loadDialog() {
  const mod = await import('./dialog.js');
  mod.show();
}
```
### 典型使用场景
1. 路由懒加载（Vue/React 分包）；
2. 条件加载、按需加载大组件；
3. 循环内动态导入不同模块；
4. 加载路径动态拼接。
### 特性
- 属于运行时 API，可放在 `if/for` 内部；
- 支持传入模板字符串动态拼接路径；
- 返回模块命名空间对象，包含所有导出；
- 浏览器/Node 均支持，打包工具自动代码分割。

## 39. 模块的继承
### 原理
通过 `export * from` 实现模块继承，子类模块导入父模块所有接口，再扩展自身方法。
```js
// base.js 父模块
export const baseFn = () => {};

// child.js 继承模块
export * from './base.js'; // 继承所有导出
// 扩展自有方法
export const childFn = () => {};

// 使用
import { baseFn, childFn } from './child.js';
```
### 覆盖父模块接口
同名导出会覆盖继承来的父模块变量：
```js
export * from './base.js';
export const baseFn = () => {
  console.log('重写父方法');
};
```
### 默认导出继承补充
`export * from` 不会继承 default，如需继承默认导出要单独写：
```js
export * from './base.js';
export { default } from './base.js';
```

## 40. ES6 Module 浏览器加载
### 1. script 标签开启模块模式
```html
<!-- 必须加 type="module" -->
<script type="module" src="./main.js"></script>

<!-- 内联模块 -->
<script type="module">
  import { a } from './util.js';
</script>
```
### 浏览器模块关键特性
1. **默认跨域限制**：本地打开 `file://` 协议会报错，必须启动本地服务；
2. **延迟执行**：`type="module"` 等价于加 `defer`，页面解析完成后再执行，不阻塞渲染；
3. **严格模式**：模块内自动开启 `use strict`，无需手动声明；
4. **作用域隔离**：模块顶层变量不会暴露到全局 window；
5. 支持 `import.meta.url` 获取当前模块文件路径。
### 预加载辅助
```html
<link rel="modulepreload" href="./util.js">
```
提前下载模块，提升加载速度。

## 41. ES6 模块的转码
### 核心背景
浏览器原生支持度有限、旧环境不识别 import/export，需工具转译：
### 1. Babel 转译核心问题
Babel 仅转换语法，**不会处理模块加载逻辑**：
`import / export` 只会转为 CommonJS `require/module.exports`，原生浏览器无法识别。

### 2. 配套打包工具（缺一不可）
1. **Webpack / Rollup / Vite**
   - Webpack：适配工程，转换模块 + 打包、代码分割；
   - Rollup：适合类库打包，输出标准 ES Module / CommonJS；
   - Vite：开发环境直接利用浏览器原生 ES Module，生产打包。
2. 两种输出格式
   - CJS：`module.exports / require`，适配 Node；
   - ESM：保留 `import/export`，现代浏览器直接使用。

### 3. Node.js 环境模块兼容
Node 默认是 CommonJS，识别 ESM 两种方式：
1. 文件后缀 `.mjs`；
2. package.json 添加 `"type": "module"`。

### 4. 转码完整流程
源码 ES Module → Babel 语法转换 → 打包工具解析依赖图 → 输出兼容产物（浏览器 ESM / Node CJS）

需要我把这6个知识点整理成一份极简背诵版提纲，方便复习吗？