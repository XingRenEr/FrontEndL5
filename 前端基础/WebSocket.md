# WebSocket 客户端六大核心API完整详解
WebSocket 是浏览器原生全双工通信协议，建立连接后客户端与服务端可双向实时收发数据，下面逐个讲解 `onopen / onclose / onmessage / send() / bufferedAmount / onerror`，包含作用、触发时机、参数、示例、使用场景。

## 前置基础：创建WebSocket实例
```javascript
// 新建连接，ws普通、wss加密
const ws = new WebSocket("wss://xxx.com/ws");
```
所有API都挂载在 `ws` 实例上。

---

## 1. webSocket.onopen 连接成功回调
### 作用
**WebSocket 握手完成、连接正式建立时触发**，代表可以正常收发消息。
### 触发时机
1. 客户端发起 ws 连接请求
2. 服务端返回 101 Switching Protocols 协议切换响应
3. 握手成功，立即执行 `onopen`

### 特点
- 只会执行**一次**；
- 在此回调内才能安全调用 `send()`，连接未就绪前发送会报错。

### 示例
```javascript
ws.onopen = function (event) {
  console.log("连接建立成功", event);
  // 连接成功后立即发送登录/鉴权消息
  ws.send(JSON.stringify({ type: "login", token: "xxx" }));
};
```
### event 对象
`Event` 基础事件对象，无额外业务字段，仅标记连接成功事件。

### 常用场景
连接建立后发送身份认证、订阅频道、初始化同步数据。

---

## 2. webSocket.onclose 连接关闭回调
### 作用
连接被关闭时触发，区分**主动关闭**和**异常断开**。
### 触发时机
1. 客户端手动调用 `ws.close()`
2. 服务端主动断开连接
3. 网络中断、页面关闭、服务端下线

### 回调参数 CloseEvent
包含4个关键属性：
- `code`：关闭状态码（标准规范）
- `reason`：服务端/客户端传递的关闭文本原因
- `wasClean`：布尔值，`true` = 正常关闭；`false` = 异常断线（断网、崩溃）

### 标准关闭码常用值
| code | 含义                                       |
| ---- | ------------------------------------------ |
| 1000 | 正常关闭                                   |
| 1006 | 异常断开（断网、服务宕机，无正常关闭握手） |
| 4001 | 自定义：token失效踢出                      |
| 4002 | 自定义：重复登录                           |

### 示例
```javascript
ws.onclose = function (event) {
  console.log("连接关闭", event.code, event.reason, event.wasClean);
  // 异常断线自动重连
  if (!event.wasClean || event.code === 1006) {
    setTimeout(() => reconnectWs(), 2000);
  }
};
```

### 场景
断线重连、提示用户离线、清理页面轮询/定时器、退出登录。

---

## 3. webSocket.onmessage 接收服务端消息回调
### 作用
**服务端推送消息到客户端时触发**，核心业务接收入口。
### 触发时机
服务端通过 `ws.send()` 下发任意数据，客户端立即接收。

### 回调参数 MessageEvent
核心属性 `event.data`，服务端返回的数据，类型分三种：
1. `string`：最常用，JSON字符串、普通文本；
2. `Blob`：二进制文件、图片流；
3. `ArrayBuffer`：底层二进制字节流。

### 示例（JSON业务数据）
```javascript
ws.onmessage = function (event) {
  const data = JSON.parse(event.data);
  switch (data.type) {
    case "chat":
      // 聊天消息渲染页面
      renderMsg(data.content);
      break;
    case "notice":
      showNotice(data.msg);
      break;
  }
};
```

### 二进制数据处理示例
```javascript
ws.onmessage = async (e) => {
  if (e.data instanceof Blob) {
    const arrayBuf = await e.data.arrayBuffer();
  }
};
```

### 场景
实时聊天、消息通知、实时数据大屏、直播弹幕、设备状态推送。

---

## 4. webSocket.send() 客户端发送消息方法
### 作用
客户端向服务端主动发送数据，**唯一发送API**。
### 调用前提
必须在 `onopen` 触发后调用，连接未就绪调用会抛出异常。

### 支持传入4种数据类型
1. **字符串**（业务首选，JSON序列化）
2. ArrayBuffer 二进制字节
3. TypedArray 类型数组
4. Blob 文件二进制

### 示例
```javascript
// 1. 发送JSON文本
ws.send(JSON.stringify({ type: "chat", to: "user1", text: "你好" }));

// 2. 发送二进制
const buf = new Uint8Array([1,2,3]);
ws.send(buf);
```

### 注意事项
1. 断开状态调用会报错，发送前判断连接状态 `ws.readyState === 1`；
2. 高频发送大量数据会堆积缓冲区，配合 `bufferedAmount` 限流；
3. 大文件分片发送，避免单次传输过大阻塞通道。

### readyState 连接状态枚举
- 0 CONNECTING：连接中
- 1 OPEN：可正常发送
- 2 CLOSING：关闭中
- 3 CLOSED：已关闭

安全发送封装：
```javascript
function safeSend(data) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(data);
  }
}
```

---

## 5. webSocket.bufferedAmount 发送缓冲区字节数（只读属性）
### 作用
获取**等待发送、还未传输到服务端的数据总字节大小**，用于流量限流、防发送过载。
### 原理
网络带宽有限时，频繁调用 `send()` 数据会存入浏览器发送缓冲区，`bufferedAmount` 实时统计堆积字节。

### 使用场景
高频实时场景（如游戏、画板、视频指令）限流，防止缓冲区爆堆、卡顿。

### 限流示例
```javascript
// 控制缓冲区堆积不超过10KB
function sendLimit(data) {
  if (ws.bufferedAmount > 10240) {
    console.warn("发送缓冲区过载，暂停发送");
    return;
  }
  ws.send(data);
}
```

### 特点
- 只读，不能赋值；
- 发送完成后数值自动下降；
- 单位：字节 Byte。

---

## 6. webSocket.onerror 通信异常回调
### 作用
连接发生任意错误时触发：网络错误、握手失败、协议错误、跨域、证书错误等。
### 触发时机
1. ws地址无法连接（域名错误、端口不通）
2. wss证书失效
3. 跨域拦截
4. 传输过程中网络中断
5. 协议报文非法解析失败

### 关联逻辑
`onerror` 触发后，**一定会紧接着执行 onclose**，error 本身不会携带关闭码，断线逻辑统一放在 `onclose` 处理。

### 回调参数 ErrorEvent
包含 `message` 错误描述、`error` 原始错误对象。

### 示例
```javascript
ws.onerror = function (err) {
  console.error("WebSocket通信异常：", err.message);
  alert("实时连接出错，请检查网络");
};
```

### 区分 onerror 和 onclose
- `onerror`：单纯报错事件，只记录错误信息；
- `onclose`：连接终结事件，统一处理断线、重连逻辑；
业务代码不要在onerror里写重连，避免重复执行。

---

# 完整可运行综合示例（整合全部API）
```javascript
// 创建ws连接
const ws = new WebSocket("wss://echo.websocket.org");

// 1. 连接成功
ws.onopen = (e) => {
  console.log("连接建立", e);
  ws.send(JSON.stringify({ action: "init" }));
};

// 2. 接收消息
ws.onmessage = (e) => {
  const res = JSON.parse(e.data);
  console.log("收到服务端消息：", res);
};

// 3. 缓冲区监控
setInterval(() => {
  if (ws.bufferedAmount > 5000) {
    console.log("发送缓冲区堆积：", ws.bufferedAmount, "字节");
  }
}, 1000);

// 4. 发送封装
function sendMsg(obj) {
  if (ws.readyState !== WebSocket.OPEN) return;
  // 缓冲区限流
  if (ws.bufferedAmount > 10240) return;
  ws.send(JSON.stringify(obj));
}

// 5. 通信错误
ws.onerror = (err) => {
  console.error("WS异常", err);
};

// 6. 连接关闭
ws.onclose = (e) => {
  console.log("连接关闭", e.code, e.reason, e.wasClean);
  // 异常断线自动重连
  if (!e.wasClean) setTimeout(() => location.reload(), 3000);
};

// 页面卸载主动关闭连接
window.onbeforeunload = () => {
  ws.close(1000, "用户离开页面");
};
```

# 六大API核心总结
| API            | 类型     | 核心用途               | 触发时机                      |
| -------------- | -------- | ---------------------- | ----------------------------- |
| onopen         | 事件回调 | 连接就绪，初始化发送   | 握手成功一次触发              |
| onmessage      | 事件回调 | 接收服务端所有推送消息 | 服务端下发数据时              |
| send()         | 方法     | 客户端主动发数据       | 仅onopen后可用                |
| bufferedAmount | 只读属性 | 监控发送缓冲区，限流   | 随时读取                      |
| onerror        | 事件回调 | 捕获网络/协议异常      | 通信出错立刻触发              |
| onclose        | 事件回调 | 处理断线、重连、清理   | 连接彻底关闭（error后必触发） |