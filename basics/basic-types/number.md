## 数值类型(number)

### 整型(integer)

Go 提供了 11 种整型，其中包括 5 种有符号的和 5 种无符号的，还有 1 种用来保存指针的整型类型

| 类型 | 取值范围 |
| :--- | :---- |
| int | 不同平台下的实现不同，可能是 int32 或 int64 |
| int8 | [-128, 127] |
| int16 | [-32 768, 32 767] |
| int32 | [-2 147 483 648, 2 147 483 647] |
| int64 | [-9 223 372 036 854 775 808, 9 223 372 036 854 775 807] |
| uint | 不同平台下的实现不同，可能时 uint32 或 uint64 |
| uint8 | [0, 255] |
| uint16 | [0, 65 535] |
| uint32 | [0, 4 294 967 295] |
| uint64 | [0, 18 446 744 073 709 551 615] |
| `rune` | 等同于 int32，按惯例，用于区别字符值和 int32，即需要表示一个字符值时，使用 rune 来代替 int32 |
| `byte` | 等同于 uint8，按惯例，用于区别字节值和 uint8，即需要表示一个字节数值时，使用 byte 来代替 uint8 |
| `uintptr`| 未指定大小，但足够容纳任何指针的位模式 |

> 有符号整型采用 2 的补码形式表示，高位表示符号位；一个 n 位有符号数的取值范围是： $-2^{n-1}$ ~ $2^{n-1}-1$     
> 无符号整数的所有 bit 位都表示非负数；一个 n 位无符号数的取值范围是：$0 ~ 2^{n}-1$

每种数值类型都不同，所以不同数值类型之间不能直接进行二进制数值运算或者比较操作

> 无类型的数值常量，可以兼容表达式中任何内置类型的数值

若需要在不同数值类型之间进行数值运算或比较操作，必须先进行类型转换，通常将类型转换成较大的类型以防止精度丢失。

> 类型转换采用 type(value) 的形式

标准库中提供了 `big.Int` 类型的整数和 `big.Rat`（处理财务计算时特别有用） 类型的有理数，这些都是不限大小的（只限于机器的内存）

#### 位运算

| 操作符 | 说明 | 规则 | 适用对象|
| :--- | :---- | :---- | :---- |
| & | 位运算 AND | 同位都是 1，则该位取 1，否则取 0 | 有/无符号数 |
| \| | 位运算 OR | 同位都是 0，取 0。也就是说同位只要有一个 1，则取 1 | 有/无符号数 |
| ^ | 位运算 XOR |TODO | 有/无符号数 |
| &^ | 位清空 AND NOT | 根据左边的数，按位相异的保留左边位，相同的位清零 | 有/无符号数 |
| << | 左移 | TODO | 无符号数 |
| >> | 右移 |TODO | 无符号数 |



参考：https://blog.csdn.net/yjk13703623757/article/details/83958320?utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.no_search_link



位运算符规则说明：操作符左边的数和右边的数按二进制位进行计算

`&`

`|`

`^`

`&^` 示例：

```
0001 0100 &^ 0000 1111 = 16

   0001 0100    = 20
&^ 0000 1111    = 15
-------------
   0001 0000    = 16
```



`<<`

`>>`





##### 位运算的使用



- `x & (x -1)` ：将 x 的最右侧的 1 变为 0

    - 判断一个整数是否是 2 的 n 次方: `x & (x-1) == 0`

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

    
    - 计算统计一个正整数的二进制中包含多少个 1

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

    

### 浮点型

浮点数表示的是近似值，当需要高精度计算时，可以使用 `big.Int` 和 `big.Rat` 

Go 种提供了 2 种类型的浮点数类型：`float32` 和 `float64`

> Golang浮点型的默认舍入规则——四舍六入五成双 https://studygolang.com/articles/8018

Go 中使用 [IEEE-754](http://em.wikipedia.org/wiki/IEEE_754-2008) 格式表示浮点数，该格式也是很多处理器以及浮点数单元所使用的原生格式

浮点数的范围极限值可以在 math 包中找到。math.MaxFloat32 表示 float32 能表示的最大值; math.MaxFloat64 表示 float64 能表示的最大值

| 类型 | 取值范围 | 计算精度 |
| :--- | :---- | :---- |
| float32 | SmallestNonzeroFloat32 = 1.401298464324817070923729583289916131280e-45 // 1 / 2**(127 - 1 + 23) </br> MaxFloat32 = 3.40282346638528859811704183484516925440e+38  // 2**127 * (2**24 - 1) / 2**23 | 大约 6 个十进制数 |
| float64 | SmallestNonzeroFloat64 = 4.940656458412465441765687928682213723651e-324 // 1 / 2**(1023 - 1 + 52) </br> MaxFloat64 = 1.797693134862315708145274237317043567981e+308 // 2**1023 * (2**53 - 1) / 2**52 | 大约 15 个十进制数 |

当使用浮点数时，应优先使用 float64 类型，因为 float32 类型的累计计算误差很容易扩散，且 float32 类型能表示的正整数不是很大（float32 的有效 bit 位只有 23 位，其他的 bit 位用于指数和符号；当整数部分大于 23 bit 能表达的范围时，将出现误差）：

```go
var f float32 = 16777216 // 1 << 24
fmt.Println(f == f+1) // true
```

### 复数

Go 提供了 2 种类型的复数类型：`complex64` 和 `complex128`，分别对应 `float32` 和 `float64` 两种浮点精度

### 算术操作符

