## 常量（constants）

> 参见：https://go.dev/ref/spec#Constants

常量有 布尔常量，字符常量，整型常量，浮点常量，复数常量和字符串常量。其中 字符/整型/浮点型/复数常量统称为 **数值型** 常量

bool 真值由预声明的常量 `true` 和 `false` 表示

预声明的标识符 `iota` 表示整数常量

一般，复数常量是一种常量表达式形式，这个在[常量表达式](TODO)这一节介绍


数值型常量代表任意精度的精确值，不会溢出。因此没有常量表示 IEEE-754 负零，无穷大和非数字值


常量可能是有类型或无类型的。**字面量常量，true，false，iota，和某些仅包含无类型常量操作数的常量表达式是无类型的**


一个常量可能是类型明确的或类型不明确的：

- 类型明确的：通过常量声明或者类型转换得到的类型明确的常量
- 类型不明确的：在变量声明或者赋值语句中使用或作为表达式中的操作数隐式赋予类型

若常量值不能表示相应类型的值，则为错误。若类型是类型参数，则常量将转换为类型参数的非常量值

无类型常量具有默认类型，即常量在需要类型值的上下文中隐式转换为的类型，例如，在没有显式类型的短变量声明中，如 `i := 0`。无类型常量的默认类型分别是 bool, rune，int, float64, complex128, 或 string，这取决于它是布尔、字符、整型、浮点型、复数还是字符串常量

实现限制：虽然数值常量在语言中具有任意精度，但编译器可能使用精度有限的内部表示来实现它们，尽管如此，每个实现必须：

- 表示整型常量至少 256 位
- 表示浮点型常量，包括复数常量部分，至少 256 位的尾数和至少 16 位的有符号二进制指数来表示浮点常量
- 若不能表示整数常量，则给出错误
- 若由于溢出无法表示浮点和复数常量，则给出错误
- 若由于精度限制无法表示浮点和复数常数，则近似到最接近的可表示常数

这些要求既适用于字面常量，也适用于评估常量表达式的结果

### 声明和初始化

- 常量与变量一样声明，但使用 `const` 关键字
- 常量可以是 字符/字符串/布尔值/数值类型
- 常量不能使用短变量声明语法 `:=` 

```go
package main

import "fmt"

const Pi = 3.14

func main() {
	const World = "世界"
	fmt.Println("Hello", World)
	fmt.Println("Happy", Pi, "Day")

	const Truth = true
	fmt.Println("Go rules?", Truth)
	
	/*
	-- output：
	  Hello 世界
	  Happy 3.14 Day
	  Go rules? true
	*/
}
```

- 在定义常量组时，如果不提供初始化值，则表示将使用上一行的表达式
- 使用相同的表达式不代表具有相同的值
- itoa是常量的计数器，从 0 开始，组中每定义 1 个常量，则itoa自动递增 1
- 通过初始化规则与 iota 可以达到枚举的效果
- 每遇到一个 const 关键字， iota 就会重置为 0


```go
const(
	// a 和 b 都是"A"
	a = "A"
	b
	c = itoa	// 2
	d 			// 3
	// itoa 是常量的计数器，并不是每使用一次才 +1
)

const(
	e = iota
	f       // f = 1
)

const(
	// 第一个常量不可省略
	Mondy = iota
	Tuesday
	Wednesday
)
```



