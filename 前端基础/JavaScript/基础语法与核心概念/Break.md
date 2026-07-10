# JS break 关键字详解
## 一、作用
1. **终止当前整个循环**（`for / while / do while`），跳出循环，执行循环后面代码
2. 在 `switch` 中：终止 case，防止穿透

## 二、break 在循环里
### 1. for 循环
```js
for(let i = 1; i <= 5; i++){
  if(i === 3){
    break; // 直接结束循环
  }
  console.log(i);
}
// 输出：1 2
```

### 2. while 循环
```js
let i = 1;
while(i <= 5){
  if(i === 3) break;
  console.log(i);
  i++;
}
// 输出：1 2
```

### 3. do while
```js
let i = 1;
do{
  if(i === 3) break;
  console.log(i);
  i++;
}while(i <= 5);
```

## 三、break 在 switch 中（必用）
不加 break 会**case穿透**，匹配后继续执行后面所有case
```js
let num = 2;
switch(num){
  case 1:
    console.log(1);
    break;
  case 2:
    console.log(2);
    break; // 跳出switch
  default:
    console.log("其他");
}
```

## 四、标签 label + break（跳出多层循环）
默认 `break` 只能跳出**最内层**循环，想要一次性跳出多层，给外层循环加标签：
```js
outer: for(let i = 1; i <= 3; i++){
  for(let j = 1; j <= 3; j++){
    if(i === 2 && j === 2){
      break outer; // 直接跳出外层标记的循环
    }
    console.log(i,j);
  }
}
```

## 五、和 continue 区分
- `break`：彻底结束整个循环
- `continue`：仅跳过当前这一轮，继续下一轮循环
```js
for(let i=1;i<=4;i++){
  if(i===2) continue;
  console.log(i); // 1 3 4
}
```

## 六、使用限制
`break` 只能出现在：
- for / while / do while 循环体内
- switch case 代码块内

写在普通if、函数外层会直接报错：
```js
if(true){
  break; // 语法错误
}
```

## 七、常见场景
1. 找到目标数据，无需继续循环，直接退出
2. switch 阻断case穿透
3. 多层嵌套循环需要一次性全部跳出（搭配label标签）