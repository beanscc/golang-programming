## reflect

> 原文: https://blog.golang.org/laws-of-reflection

## Introduction

反射在计算中是程序检查自己数据结构的一种能力，特别是通过类型；它是元编程的一种形式，同时也是混乱的一个重要来源。

在这篇文章中我们将尝试通过解释反射在 Go 中是如何工作的来澄清一些事儿。每种语言的反射模型是不同的（也有很多语言不支持反射），但该篇我们只关注 Go 的反射，因此下文中所提到的“反射”，皆表示“Go中的反射”。

## Types and interfaces

因为反射是构建在语言的类型系统上的，所以我们接下来先复习一下 Go 的数据类型。

Go 是静态类型的。每个变量都有一个静态类型，在编译期是一个已知的、固定的类型：int, float32, *MyType, []byte 等等。若我们定义

```go
type MyInt int

var i int
var j MyInt
```

这里 `i` 的类型是 `int`, `j` 的类型是 `MyInt`。变量 `i` 和 `j` 有不同的静态类型，尽管它们有相同的底层类型，它们之间在未经类型转换时也不能相互赋值。

interface 类型是类型中一个重要的分类，它代表了方法的固定集合。一个 interface 变量可以存储任何具体值（非 interface ），只要这个值实现了 interface 定义的方法。
一对比较有名的示例是 `io.Reader` 和 `io.Writer`, 它们来源于 [io package](https://golang.org/pkg/io)

```go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
	Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

任何类型实现了以上签名参数的Read(或 Write) 方法，则就实现了 `io.Reader` (or `io.Writer`) 接口。这里讨论的目的，意味着一个 `io.Reader` 类型的变量，可以存储任何实现了 Read 方法的值：

```go
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
// and so on
```

重要的是要清楚 r 存储的无论是何种具体的值， r 的类型总是 `io.Reader`：Go 是静态类型的，并且 r 的静态类型是 `io.Reader`

一个非常重要的 interface 类型的例子是，空接口：

```go
interface{}
```

它代表了方法的空集合，并且任何值都满足该接口，因为任何值都拥有零个或多个方法。

有些人说 Go 的 interface{} 是动态的类型，这是误导。
interface 是静态类型的：一个 interface 类型的变量在声明之后始终是该静态类型，即使在运行时该接口变量存储的值可能会改变其类型，但其值也总是满足接口的。

我们需要对所有这一切都保持精确，因为反射和 interface 密切相关

## The representation of an interface

Russ Cox 写过一篇关于[Go interface 值的表示](https://research.swtch.com/interfaces)的详细博客。这里没有必要重复完整内容，但简介的概要还是有必要的

一个 interface 类型的变量存储了一对：赋值给这个变量的具体值；这个值的类型描述符。更确切的讲，这个值是实现了该 interface 接口的底层具体数据项，并且值的类型描述了数据项的完整类型。举个例子：

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
	return nil, err
}
r = tty
```

r 示意性地包含（值，类型）对,（tty, *os.File）。注意 *os.File 类型实现了不仅仅是 Read 方法；即使 interface 变量值 r 仅提供对 Read 方法的访问，但变量值内部的值仍包含了关于该值的全部类型信息。这就是为什么我们可以这样做：

```go
var w io.Writer
w = r.(io.Writer)
```

这个赋值语句中的表达式是一个类型断言；它所断言的是 r 内部的数据项也实现了 io.Writer 接口，所以我们可以将其赋值给 w。赋值语句结束后，w 将包含（tty, *os.File）对。
这和 r 所持有的对一样。interface 的静态类型决定了一个 interface 变量可以使用哪些方法，即使内部的具体值可能有一个更大的方法集合。

继续，我们可以这样做：

```go
var empty interface{}
empty = w
```

上面空接口变量值 `empty` 将再次包含同样的（tty, *os.File）对。这很方便：一个空接口可以保存任何值，并包含我们可能需要的有关该值的所有信息。

（我们这里不需要类型断言，因为静态地知道 w 满足空接口。在上面例子中，我们将一个值从 Reader 接口移动到 Writer 接口，我们需要显式地使用类型断言，因为 Writer 接口方法集不是 Reader 接口方法集的子集）

一个重要的细节是 interface 内部始终具有（值，具体类型）形式的对，且不能有（值，接口类型）形式的对。interface 不保存 interface 值

现在我们可以开始反射了

## The first law of reflection

**1. Reflection goes from interface value to reflection object.**

在基础层面，反射仅仅是一个用来检查保存在 interface 变量内部的类型和值的机制。开始学习前，这里需要我们了解[反射包](https://golang.org/pkg/reflect/)中 2 个类型：[Type](https://golang.org/pkg/reflect/#Type) 和 [Value](https://golang.org/pkg/reflect/#Value)。这 2 个变量提供了访问 interface 变量内容的入口，有 2 个简单的函数 `reflect.TypeOf` 和 `reflect.ValueOf` 分别从 interface 变量中返回 `reflect.Type` 和 `reflect.Value`。（同样，从 `reflect.Value` 中获取 `reflect.Type` 也很容易，但从现在让我们保持 `Value` 和 `Type` 的概念独立）

让我们从 `TypeOf` 开始：

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var x float64 = 3.4
	fmt.Println("type:", reflect.TypeOf(x))
}
```

这段程序输出：
```bash
type: float64
```

你可能好奇这里哪里有 interface，因为程序看起来是传递了一个 `float64` 类型的变量 `x`，而不是 interface 值给 `reflect.Typeof`函数。但就是这里；正如 [godoc reports](https://golang.org/pkg/reflect/#TypeOf) 描述的一样，`reflect.TypeOf` 函数的签名参数包括一个 `空接口`（empty interface）:

```go
// TypeOf returns the reflection Type of the Value in the interface{}.
func TypeOf(i interface{}) Type
```

当我们调用 `reflect.TypeOf(x)` 时, `x` 首先被保存到一个空接口中，然后这个空接口将被作为参数传递给 `TypeOf` 。 `reflect.TypeOf` 解开这个空接口以恢复类型信息。

当然，`reflect.ValueOf` 函数可以恢复值（从这里开始，我们将省略示例代码并只关注可执行的代码）

```go
var x float64 = 3.4
fmt.Println("value:", reflect.ValueOf(x).String())
```

输出

```
value: <float64 Value>
```

（我们显式地调用了 String() 方法，因为默认情况下 fmt 包会深入 reflect.Value 内部以展示具体的值。String() 方法将不会这样）

`reflect.Type` 和 `reflect.Value` 都有很多方法可以让我们检查和操作它们。一个重要的示例是 `Value` 有一个 `Type` 方法，其将返回该 `reflect.Value` 的 `Type` 信息 。另一个重要示例是 `Type` 和 `Value` 都有 `Kind()` 方法，其将返回一个用于标识是哪种类型项的常量：`Uint`、`Float64`、`Slice` 等。同样 `Value` 的一些方法，如 `Int()` 和 `Float()` 使我们可以获取到其内部保存的值（如 `int64` 、`float64`）:

```go
var x float64 = 3.4
v := reflect.ValuOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```

输出：
```
type: float64
kind is float64: true
value: 3.4
```

也有如 `SetInt()` 和 `SetFloat()` 这样的方法，但要使用它们我们必须理解`可设置性`（反射的第三定律的主题，将在后面讨论）

反射库具有几个值得一提的属性。

**第一：** 保持API简洁，`Value` 的 getter 和 setter 方法作用在可以保存该值的更大的类型上：所有整型的签名都是 `int64`，举个例子，`Value` 的 `Int()` 方法返回 `int64` 类型，`SetInt()` 方法接收一个 `int64` 类型的参数；转换成实际类型可能是必要的：
```go
var x uint8 = 'x'
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())                            // uint8
fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true
x = uint8(v.Uint())                                       // v.Uint() returns a uint64
```

**第二：** 反射对象的 `Kind()` 方法描述了其底层数据类型，而非静态类型。若一个反射对象包含一个用户定义的整型类型值，如下：
```go
type MyInt int
var x MyInt = 7
v := reflect.ValueOf(x)
fmt.Printf("type: %v, kind: %v", v.Type(), v.Kind())  // type: main.MyInt, kind: int
```
则 v 的 `Kind()` 方法返回的依然是 `reflect.Int`，即使变量 x 的静态类型是 `MyInt`，而非 `int`。换句话说，即使 `Type()` 可以, `Kind()` 方法不能将 `MyInt` 和 `int` 区别开。

## The Second Law of reflection

**2. Reflection goes from reflection object to interface value.**

像物理反射一样，Go 中的反射会生成自己的逆。

给定一个 `reflect.Value` 我们可以使用 `Interface()` 方法来恢复一个 interface 值，事实上这个方法将类型和值信息打包到一个 interface 表示形式并作为结果返回：

```go
// Interface returns v's value as an interface{}.
func (v Value) Interface() interface{}
```

从结果上我们可以说

```go
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```

将打印反射对象 v 所代表的 float64 值

尽管我们可以做的更好。fmt.Println, fmt.Printf 等的参数都是作为空接口值传递的，然后被 fmt 包内部解析，就如我们前面的例子所做的。因此正确地打印 `reflect.Value` 的内容所要做的就是传递 `Interface()` 方法的返回值给格式化打印协程：

```go
fmt.Println(v.Interface())
```

（为什么不使用 fmt.Println(v)? 因为 v 是一个 `reflect.Value` 值； 我们想要的的它保存的具体值）。既然我们的值是一个 `float64`，我们甚至可以使用浮点格式若我们需要的话：

```go
fmt.Printf("value is %7.1e\n", v.Interface())
```

这种情况下

```
3.4e+00
```

同样，这里不必进行类型断言判断 `v.Interface()` 是否是 `float64` 类型；空接口值内部有具体值类型信息，且 `Printf` 会恢复它的的类型

简单讲，`Interface()` 方法就是 `ValueOf` 函数的逆，只是它的结果总是静态类型 `interface{}`

**重申**：反射从接口值到反射对象，然后再反转 

## The third law of reflection

**3. To modify a reflection object, the value must be settable**

第三定律是最微妙但也让人疑惑的，但若从第一定律开始理解，也就很容易理解

这里有些不能正常工作的代码，但却值得研究

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1)  // Error: will panic.
```

若你执行上面的代码，将会得到如下 panic 信息

```
panic: reflect.Value.SetFloat using unaddressable value
```
问题不是值 7.1 不可寻址，而是变量 v 不是可设置的。 可设置性是 `reflect.Value` 的一个属性，且并非所有的反射值都有它

`Value` 的 `CanSet()` 方法可以表示一个 `Value` 的可设置性；在我们的例子中

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
```

输出：
```
settability of v: false
```

在不可设置的 `Value` 上调用 `SetXXX()` 方法将得到一个 err 错误。 但何为可设置性呢？

可设置性有一点像可寻址性，但更严格。反射对象可以修改实际存储这一特性，常被用来创建反射对象。反射对象是否保存原始数据项决定了可设置性。当我们讲：

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
```

传递给 `reflect.ValueOf` 的只是 x 的拷贝对象，所以作为 `reflect.ValueOf` 的参数所创建的 interface 只是 x 的拷贝，不是 x 本身。

```go
v.SetFloat(7.1)
```

因此若允许语句执行成功，它也不会更新 x，即使 v 看起来是创建自 x。相反，它将更新反射值内部存储的 x 的拷贝体，而 x 本身不会受影响。这样的话将使人疑惑且无任何用处，所以这是非法的，而可设置性就是用来避免这一问题的一个特性。

这看起来怪异吗？不是这样的。它实际上是不平常外衣下的常见情景。想象一下将 x 传递给一个函数：

```go
f(x)
```

我们不期望 f 有能力修改 x 的值，因为我们传递是 x 的拷贝值，而非 x 本身。若我们想让 f 直接修改 x，我们必须将 x 的地址传递给函数（也就是，指向 x 的指针）：

```go
f(&)
```

这是比较直接和熟悉的，反射也是一样的工作原理。若我们希望通过反射修改 x，我们必须给反射库一个指向我们想修改值的指针。

让我们就这样做吧。首先，我们像平常一样初始化 x，然后创建一个指向它的反射值，叫做 p

```go
var x float64 = 3.4
p := reflect.ValueOf(&x) // Note: take the address of x.
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())
```

到目前输出是：
```
type of p: *float64
settability of p: false
```

反射对象 p 不是可设置的，但我们并不是想设置 p，而实际上是 *p。为了获取 p 所指向的值，我们调用 `Value` 的 `Elem()` 方法，它间接通过指针，并将结果保存到名为 v 的反射 Value 中：

```go
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
```

现在 v 是一个可设置的反射对象，正如输出所示：

```
settability of v: true
```

既然它代表 x，我们终于可以使用 `v.SetFloat()` 来修改 x 的值：

```go
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
```

预期输出是：

```
7.1
7.1
```

反射可能很难理解，但它所做的正是语言所做的，尽管通过反射可以掩盖正在进行值的类型和值。请记住反射值需要寻址某些东西是为了修改其所代表的值。

## Structs

在我们之前的例子中 v 自身不是一个指针，它只是从一个指针值派生而来的。发生这种情况的常见方式是使用反射来修改结构体的字段时。只要我们有结构体的地址，就可以修改它的字段值

这里有个分析结构体值 t 简单的例子。 我们使用结构体的地址创建一个反射对象，因为我们想后面修改它的值。然后我们将它的 `Type` 设置为 `typeOfT`, 并使用简单的方法调用对每个字段进行迭代（具体见 [reflect](https://golang.org/pkg/reflect/) 包）。注意我们从结构体的类型中把字段名解析出来了，但字段本身还是一个常规的 `reflect.Value` 对象。

```go
type T struct {
	A int
	B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
	f := s.Field(i)
	fmt.Printf("%d: %s %s = %v\n", i,
	typeOfT.Field(i).Name, f.Type(), f.Interface())
}
```

程序输出是：

```
0: A int = 23
1: B string = skidoo
```

还有一点关于可设置性介绍的：结构体 T 的字段名称是大写开始的（即可导出的），因为只有可导出的字段才可设置

因为 s 包含一个可设置的反射对象，所以我们可以修改其中的字段

```go
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)
```

输出结果是：
```
t is now {77 Sunset Strip}
```

若我们修改程序使 s 创建自 t 而非 &t，那么调用 `SetInt()` 和 `SetString()` 将失败，因为 t 的字段不可设置。

## Conclusion

再次重申反射的几大定律：
- 反射从接口值到反射对象
- 反射从反射对象到接口值
- 要修改反射对象，则其必须是可设置的

一旦理解了 Go 中反射这几大定律，使用起来就更容易了，尽管它依然很微妙。它是一个强大的工具，使用时需要格外小心，并避免使用，除非绝对必要。

我们还有很多关于反射的没有覆盖到 -- 在 channel 上的发送和接收，内存分配，slice 和 map 的使用，方法和函数的调用 -- 但是这篇文章也足够长了。我们将在后期的文章中介绍其中一些主题。