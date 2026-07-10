# JS switch 完整详解
## 1. 基础语法
```js
switch(表达式) {
  case 匹配值1:
    执行代码;
    break;
  case 匹配值2:
    执行代码;
    break;
  default:
    所有case都不匹配时执行;
}
```

### 核心规则
1. 匹配使用 **严格相等 `===`**，不自动类型转换
2. `break`：跳出switch，不加会发生**case穿透**
3. `default` 可选，放任意位置都最后执行，建议写末尾

## 2. 基础示例
```js
let week = 3;
switch (week) {
  case 1:
    console.log("周一");
    break;
  case 2:
    console.log("周二");
    break;
  case 3:
    console.log("周三");
    break;
  default:
    console.log("休息日");
}
// 输出：周三
```

## 3. case 穿透（不写break）
多个值执行同一段逻辑，利用穿透简化代码
```js
let num = 2;
switch (num) {
  case 1:
  case 2:
  case 3:
    console.log("1~3之间");
    break;
  case 4:
    console.log("数字4");
    break;
}
// 输出：1~3之间
```

## 4. 严格匹配坑（不会隐式转换）
```js
let n = "2";
switch (n) {
  case 2:
    console.log("匹配数字2"); // 不执行，字符串 !== 数字
    break;
  case "2":
    console.log("匹配字符串2"); // 执行
}
```

## 5. switch 做区间判断（if替代思路）
switch 默认只能等值匹配，不能直接写 `case x < 60`
解决：switch括号内写 `true`，让每个case返回布尔值
```js
let score = 75;
switch (true) {
  case score >= 90:
    console.log("优秀");
    break;
  case score >= 60:
    console.log("及格");
    break;
  default:
    console.log("不及格");
}
```

## 6. default 放前面也生效
匹配顺序不变，匹配成功后break才停止
```js
let a = 5;
switch (a) {
  default:
    console.log("无匹配");
    break;
  case 5:
    console.log("等于5");
}
// 输出：等于5
```

## 7. switch 返回值写法（简化）
利用代码块+return，不用大量break
```js
function getText(type) {
  switch (type) {
    case 1: return "新增";
    case 2: return "编辑";
    case 3: return "删除";
    default: return "未知操作";
  }
}
console.log(getText(2)); // 编辑
```

## 8. 常见易错点
1. 漏写 `break`，代码一路穿透执行所有case
2. 混淆 `==` 和 `===`，字符串数字匹配不上
3. case 后不能写区间表达式（直接 `case n>10` 报错）
4. switch 括号内只能放一个表达式，不能多条件

## 9. switch vs if 选用场景
- 适合switch：固定枚举值匹配（状态码、星期、类型标识）
- 适合if else：区间判断、多复杂逻辑条件
- 