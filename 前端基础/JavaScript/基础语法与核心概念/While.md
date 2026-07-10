# JS while / do while 循环完整教程
## 一、while 循环（先判断，后执行）
### 语法
```js
while (条件表达式) {
  循环体代码
}
```
执行逻辑：
1. 先判断括号内条件，为 `true` 才执行循环体
2. 执行完循环体，回到条件再次判断
3. 条件为 `false` 直接结束循环

### 基础示例：输出 0~4
```js
let i = 0;
while (i < 5) {
  console.log(i);
  i++; // 一定要更新变量，否则死循环
}
```

### 死循环（慎用）
条件永远为 true，程序卡死
```js
while (true) {
  console.log("无限执行");
}
```

## 二、do while 循环（先执行，后判断）
### 语法
```js
do {
  循环体代码
} while (条件表达式);
```
核心特点：**至少执行一次**，不管条件真假
1. 先执行一次大括号内代码
2. 再判断 while 条件，true 继续循环，false 停止

### 示例
```js
let i = 10;
do {
  console.log(i);
  i++;
} while (i < 5);
// 输出 10，只执行一次
```

## 三、循环控制关键字（while / do while 通用）
1. `break`：直接终止整个循环
```js
let i = 0;
while (i < 10) {
  if (i === 3) break;
  console.log(i); // 0 1 2
  i++;
}
```
2. `continue`：跳过本次循环，直接进入下一轮判断
```js
let i = 0;
while (i < 5) {
  i++;
  if (i === 2) continue;
  console.log(i); // 1 3 4 5
}
```

## 四、while 和 for 的区别
1. `for`：已知循环次数、需要计数自增时首选
2. `while`：循环次数不确定，只知道终止条件时用
示例：输入数字直到输入0停止
```js
let num;
while (num !== 0) {
  num = Number(prompt("请输入数字，输入0退出"));
}
```

## 五、while vs do while 对比
| 循环     | 判断时机   | 是否至少执行一次        |
| -------- | ---------- | ----------------------- |
| while    | 循环前判断 | 条件false则一次都不执行 |
| do while | 循环后判断 | 无论条件，必然执行1次   |

## 六、常见坑
1. 忘记更新循环变量，造成死循环
```js
// 错误：i 永远不变，无限循环
let i = 0;
while (i < 5) {
  console.log(i);
}
```
2. do while 末尾必须加分号 `;`
3. 条件判断注意真假值，`null/undefined/0""` 都会终止循环