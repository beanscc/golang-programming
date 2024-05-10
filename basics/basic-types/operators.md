## Operators

文档：https://go.dev/ref/spec#Operators

golang 支持以下几类运算符：

- 算术运算符
    - 标准算术运算符  (`+`, `-`, `*`, `/`) 和 求余 (`%`)
    - 位逻辑运算符
    - 位移运算符
- 比较运算符
- 逻辑运算符



### 算术运算符

算术运算符应用于数值，***运算结果与第一个操作数类型相同***

#### 标准算术运算符


| 运算符        | 说明   | 适用类型 |
| :---------------- | :----- |:----- |
| +             | 加 | integers, floats, complex values<br />strings 字符串连接 |
| - | 减 |integers, floats, complex values|
| *             | 乘 |integers, floats, complex values|
| / | 除 |integers, floats, complex values|
| % | 求余 |integers|



#### 位逻辑运算符

运算结果：integers

| 运算符 | 说明               | 适用类型 |
| :----- | :----------------- | -------- |
| &      | 按位与 AND         | integers |
| \|     | 按位或 OR          | integers |
| ^      | 按位异或 XOR       | integers |
| &^     | 按位清除 (AND NOT) | integers |



`&^` 示例：

```
0001 0100 &^ 0000 1111 = 16

   0001 0100    = 20
&^ 0000 1111    = 15
-------------
   0001 0000    = 16
```



#### 位移运算符

运算结果：integers

| 运算符 | 说明     | 适用类型 |
| :----- | :------- | -------- |
| <<      | 左移   | integer << integer >= 0 |
| >>    | 右移   | integer >> integer >= 0 |



TODO 示例



### 比较运算符

运算结果： bool 类型

| 运算符 | 说明     | 适用类型 |
| :----- | :------- | -------- |
| ==      | 等于   | comparable |
| !=    | 不等于   | comparable |
| <    | 小于   | ordered |
| <=    | 小于或等于   | ordered |
| >    | 大于   | ordered |
| >=    | 大于或等于   | ordered |



### 逻辑运算符

运算结果：bool 类型

| 运算符            | 说明   |   适用类型   |
| :---------------- | :----- | ---- |
| &&              | 逻辑与 |   bool   |
| \|\| | 逻辑或 |  bool    |
| !               | 逻辑非 |  bool    |



golang 的逻辑运算是一个短路逻辑，什么是短路逻辑呢？简单说就是若左侧操作数已可以判定逻辑运算的结果，则不再就算右侧的操作数，具体如下：

表达式 `a1 || a2` ，且 a1 的值为 `true`；那么无论表达式 a2 的值是什么，表达式的结果都为 `true`，因此， a2 的值不会计算而是直接返回 `true`；        
表达式 `b1 && b2` ，且 b1 的值为 `false`；那么无论表达式 b2 的值是什么，都不会再计算它的值，而直接返回 `false`；





## 示例

### x & (x -1)

`x & (x -1)` 将 x 的最右侧的 1 变为 0

#### 判断一个整数是否是 2 的 n 次方



```go
package main

import "fmt"

func main() {
	x := 12
	fmt.Printf("x:%d, 是否是偶数：%t\n", x, isEven(x))
	x = 0
	fmt.Printf("x:%d, 是否是偶数：%t\n", x, isEven(x))
	x = 4
	fmt.Printf("x:%d, 是否是偶数：%t\n", x, isEven(x))
	x = 6
	fmt.Printf("x:%d, 是否是偶数：%t\n", x, isEven(x))
	x = 3
	fmt.Printf("x:%d, 是否是偶数：%t\n", x, isEven(x))
	x = 7
	fmt.Printf("x:%d, 是否是偶数：%t\n", x, isEven(x))
}

// 是否 2 的 n 次方
func isEven(x int) bool {
	return x&(x-1) == 0
}
/* output:
x:12, 是否是偶数：false
x:0, 是否是偶数：true
x:4, 是否是偶数：true
x:6, 是否是偶数：false
x:3, 是否是偶数：false
x:7, 是否是偶数：false
*/
```



#### 统计一个正整数的二进制中包含多少个 1



```go
package main

import "fmt"

func main() {
	x := 0b101011 // 0b 表示二进制表示
	fmt.Printf("x:%b, 二进制表示中 1 的个数：%d", x, count1InBinary(x))
}

func count1InBinary(x int) int {
	n := 0
	for {
		if x > 0 {
			x = x & (x - 1)
			n++
		} else {
			break
		}
	}
	return n
}
// output: x:101011, 二进制表示中 1 的个数：4
```

