## 字符串(string)

Go 中字符串是一个不可变的字节序列，实际上就是一个只读字节切片，但一般用来包含人类可读的文本。
文本字符串通常被解释为 UTF-8 编码的 Unicode 码点（rune）序列

对于英文字符串，每个字符占 8 位

字符串中的单个字符不可以被字节索引（字符串中只包含 7 位的 ASCII 字符除外）

### 字符串创建

字符串使用两种方式来创建：
- 使用双引号 `"`
- 使用反引号 <code>`</code>

```go
var s string

// 双引号方式创建
s = "hello world"

// 反引号方式创建
s = `你好，世界`
```

使用双引号创建的字符串中可以包含转义字符

使用反引号创建的字符串表示原生字符串，不包含转义字符，所有字符都是字面含义，可以包含换行\斜杠\单双引号等，一般用于创建json/xml等格式字符串，或模版字符串。但其中不能包
含反引号`` ` ``（若需要可以使用 `+` 进行字符串拼接）

### 长度

- 内置函数 len(str) 返回字符串str的字节长度，而非字符长度
- utf8.RuneCountInString(str) 返回字符串 str 的字符长度

```go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	s := "hello, 世界"
	fmt.Println(len(s))                    // 13
	fmt.Println(utf8.RuneCountInString(s)) // 9
}
```

### 字符串连接

- 使用 `+`
- strings.Join() 连接字符串切片
- 使用bytes.Buffer 或 strings.Builder

以上三种方式，效率上依次提高


```go
func main() {
	// + 方式连接字符串
	s := "hello"
	s += ", "
	s += "world"
	fmt.Println(s) // "hello, world"

	// strings.Join 方式拼接字符串
	ss := []string{"It's", "happy", "to", "learn", "golang"}
	s1 := strings.Join(ss, " ")
	fmt.Println(s1) // "It's happy to learn golang"

	// bytes.Buffer
	var buf bytes.Buffer
	buf.WriteString("Golang")
	buf.WriteByte(' ')
	buf.WriteRune('使')
	buf.WriteByte(' ')
	buf.Write([]byte("我"))
	buf.WriteByte(' ')
	buf.WriteString("efficient")
	s2 := buf.String()
	fmt.Println(s2) // "Golang 使 我 efficient"

	// strings.Builder
	var builder strings.Builder
	builder.WriteString("Builder")
	builder.WriteByte(' ')
	builder.WriteRune('可')
	builder.Write([]byte("高效的 "))
	builder.WriteString("handle string")
	s3 := builder.String()
	fmt.Println(s3) // "Builder 可高效的 handle string"
}
```

### byte

go 中使用`单引号(')`来表达`字符`，而一个字符其实就是一个与go语言所有其他整型类型兼容的整型数

`byte` 类型等于 `uint8` 类型，byte（uint8 别名）

### rune

处理关于 unicode 字符的

### utf8 和 unicode


TODO

### string/rune/byte 的转换

```go
// byte to string
var b byte
b = 'c'
bs := string(b)

// rune to byte
var r rune
r = 'c'
br := byte(r)

// rune to string
sr := string(r)
```

**单个字符转换**

- string(char): 将任何 rune 类型转换成一个包含一个字符的字符串
- []rune(s): 将一个字符串转换成一个码点切片，可索引获取单个字符

> 直接通过索引来提取字符串的字符通常是有问题的，当且仅当字符串只包含 7位 ASCII 字符的字符串时，结果才是正确的。应该使用码点切片的方式来索引。

字符串可以转成一个 Unicode 码点切片（[]rune）,这个切片是可以索引的

```go
s := "test"
sr := []rune(s) // 转换很快 O(n)
sBytes := []byte(s)  // 非常快 O(1),因为在底层 []byte 可以简单的引用字符串的底层字节而无需复制，同理 string([]byte) 原理一样
```

### 迭代遍历

TODO

### 实现原理

字符串底层数据结构(参见 `reflect.StringHeader`)

```go
// StringHeader is the runtime representation of a string.
// It cannot be used safely or portably and its representation may
// change in a later release.
// Moreover, the Data field is not sufficient to guarantee the data
// it references will not be garbage collected, so programs must keep
// a separate, correctly typed pointer to the underlying data.
// 翻译：
// StringHeader 是 string 在 runtime 的表现形式。
// 它不能被安全地或部分地使用，且它的表现形式可能在后期发布版本中改变。
// 此外，Data 字段不足以保证它所指向的数据不被gc，所以程序必须保持独立、正确地输入指向底层数据的指针。
type StringHeader struct {
	Data uintptr
	Len  int
}
```

### string 相关文章

- [Strings, bytes, runes and characters in Go](https://blog.golang.org/strings)