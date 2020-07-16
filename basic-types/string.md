## 字符串(string)

Go 中字符串是一个不可变的字节序列，实际上就是一个只读字节切片，不可修改。文本字符串通常被解释为 UTF-8 编码的 Unicode 码点（rune）序列

### 数据结构

字符串在运行时使用 `reflect.StringHeader` 结构表示字符串

```go
// StringHeader 是 string 在 runtime 的表现形式。
// 它不能被安全地或方便地使用，且它的表现形式可能在后期发布版本中改变。
// 此外，Data 字段不足以保证它所指向的数据不被gc，所以程序必须保留一个单独的、正确地输入指向底层数据的指针。
type StringHeader struct {
	Data uintptr
	Len  int
}

// 对应的私有结构
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```

字符串的不可变性:
- 字符串不可以通过数组下标访问方式修改字符串内容
- 字符串追加拼接/重新赋值，并非向原 []byte 中追加，而是创建新的[]byte，将原字符串及追加字符串 copy 到新的 []byte 中，然后将 Data 指向新的 []byte  

通过下面示例，可以看到字符串重新赋值时，内部 []byte 的变化
```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	str := "Hello, 世界"

	// str 的地址
	sh := (*reflect.StringHeader)(unsafe.Pointer(&str))
	fmt.Printf("str:%v, sh:%+v\n", &str, sh)

	// str 字符串的 Data 字段存储的值（该值是一个指针类型，所以存储了内存地址）
	sd := (*uintptr)(unsafe.Pointer(&str))
	// Data 内指针值，指向的内容
	sdv := (*[]byte)(unsafe.Pointer(sd))
	sl := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&str)) + unsafe.Sizeof(unsafe.Pointer(sd))))
	fmt.Printf("sd:%v, sd:%v, sdv:%v, sl:%v\n", sd, *sd, *sdv, *sl)

	fmt.Println("-------")
	// 重新赋值
	str = "Welcome Golang"
	sh1 := (*reflect.StringHeader)(unsafe.Pointer(&str))
	fmt.Printf("str:%v, sh:%+v\n", &str, sh1)
	sd1 := (*uintptr)(unsafe.Pointer(&str))
	sdv1 := (*[]byte)(unsafe.Pointer(sd1))
	sl1 := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&str)) + unsafe.Sizeof(unsafe.Pointer(sd1))))
	fmt.Printf("sd1:%v, sd1:%v, sdv1:%v, sl:%v\n", sd1, *sd1, string(*sdv1), *sl1)

	// output:
	/*
		str:0xc000010200, sh:&{Data:4992905 Len:13}
		sd:0xc000010200, sd:4992905, sdv:[72 101 108 108 111 44 32 228 184 150 231 149 140], sl:13
		-------
		str:0xc000010200, sh:&{Data:4993442 Len:14}
		sd1:0xc000010200, sd1:4993442, sdv1:Welcome Golang, sl:14
	*/
```

通过以上代码中 str 变量被重新赋值前后，其 reflect.StringHeader 的表示中 Data 字段所保存的 *[]byte 指针地址变化，可以看出重新赋值后
产生了新的[]byte

### 创建/赋值

字符串使用两种方式来创建/赋值：
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

### 拼接

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

### 长度

- 内置函数 len(str) 返回字符串str的字节长度，而非字符长度
- `utf8.RuneCountInString(str)` 返回字符串 str 的字符长度

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

### 字符迭代遍历

- 方式1：使用 `for range` 迭代遍历 和 字符表示

> 注意下标 i 最后几位不是连续的，所以不能使用 `for i := 0; i < len(s); i++ {` 通过下标索引位置进行迭代，直接通过索引来提取字符串
> 的字符通常是有问题的，当且仅当字符串只包含 7 位 ASCII 字符的字符串时，结果才是正确的。应该使用码点切片的方式来索引。

示例：
```go
package main

import (
	"fmt"
)

func main() {
	s := "Hello, 世界"

	fmt.Println("字节表示:", []byte(s))
	fmt.Println("通过 for range 迭代遍历 s:")
	for i, v := range s {
		fmt.Printf("i:%v, b:%v, c:%c\n", i, v, v)
	}

	fmt.Println("通过 []rune 迭代:")
	for i, v := range []rune(s) {
		fmt.Printf("i:%v, v:%v, c:%c\n", i, v, v)
	}

	fmt.Println(`错误的迭代遍历，当 s 中还有非 ASCII 码字符时，s[i]不能表示其字符:`)
	for i := 0; i < len(s); i++ {
		fmt.Printf("i:%v, b:%v, c:%c\n", i, s[i], s[i])
	}
	// output:
	/*
		字节表示: [72 101 108 108 111 44 32 228 184 150 231 149 140]
		通过 for range 迭代遍历 s:
		i:0, b:72, c:H
		i:1, b:101, c:e
		i:2, b:108, c:l
		i:3, b:108, c:l
		i:4, b:111, c:o
		i:5, b:44, c:,
		i:6, b:32, c:
		i:7, b:19990, c:世
		i:10, b:30028, c:界
		通过 []rune 迭代:
		i:0, v:72, c:H
		i:1, v:101, c:e
		i:2, v:108, c:l
		i:3, v:108, c:l
		i:4, v:111, c:o
		i:5, v:44, c:,
		i:6, v:32, c:
		i:7, v:19990, c:世
		i:8, v:30028, c:界
		错误的迭代遍历，当 s 中还有非 ASCII 码字符时，s[i]不能表示其字符:
		i:0, b:72, c:H
		i:1, b:101, c:e
		i:2, b:108, c:l
		i:3, b:108, c:l
		i:4, b:111, c:o
		i:5, b:44, c:,
		i:6, b:32, c:
		i:7, b:228, c:ä
		i:8, b:184, c:¸
		i:9, b:150, c:
		i:10, b:231, c:ç
		i:11, b:149, c:
		i:12, b:140, c:
	*/
}
```

- 方式2：通过 utf8.DecodeRuneInString() 函数迭代

````go
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	str := "Hello, 世界"

	for len(str) > 0 {
		r, size := utf8.DecodeRuneInString(str)
		fmt.Printf("c:%c size:%v\n", r, size)
		str = str[size:]
	}
	// output:
	/*
	   c:H size:1
	   c:e size:1
	   c:l size:1
	   c:l size:1
	   c:o size:1
	   c:, size:1
	   c:  size:1
	   c:世 size:3
	   c:界 size:3
	*/
}
````

### 截取

由于字符串底层数据是一个 []byte，所以可以按照 []byte 切片的操作方式来截取字符串

```go
package main

import (
	"fmt"
)

func main() {
	s := "Hello, 世界"

	hello := s[:5]
	world := s[7:]

	fmt.Printf("%s, %s", hello, world)
	// output: Hello, 世界
}
```

### string/rune/byte 的转换

- `byte`: go 中使用`单引号(')`来表达`字符`，而一个字符其实就是一个与go语言所有其他整型类型兼容的整型数; `byte` 类型等于 `uint8` 类型，byte（uint8 别名）
- `rune`: 处理关于 `unicode` 字符的

下面通过示例说明 string/byte/rune 之间如何转换：
```go
package main

import "fmt"

func main() {
	// byte -> rune/string
	fmt.Println("ASCII byte表示时的转换:")
	var b byte
	b = 'b'
	br := rune(b)
	bs := string(b)
	fmt.Printf("b:%v, br:%v, br:%c, bs:%s\n", b, br, br, bs)

	// rune -> byte/string
	fmt.Println("ASCII rune 字符表示时的转换:")
	var r rune
	r = 'c'
	rb := byte(r)
	// rune to string
	rs := string(r)
	fmt.Printf("r:%v, rb:%v, rb:%c, rs:%s\n", r, rb, rb, rs)

	fmt.Println("非 ASCII rune 字符的转换:")
	var rc rune
	rc = '世'
	rcb := byte(r)
	// rune to string
	rcs := string(rc)
	fmt.Printf("rc:%v, rcb:%v, rcb:%c, rcs:%s\n", rc, rcb, rcb, rcs)

	// string -> []byte/[]rune
	fmt.Println("文本字符串的转换:")
	var s string
	s = "hi, 世界"
	sb := []byte(s) // 非常快 O(1),因为在底层 []byte 可以简单的引用字符串的底层字节而无需复制，同理 string([]byte) 原理一样
	sr := []rune(s) // 转换很快 O(n)，转换后可通过下标索引访问单个字符
	fmt.Printf("s:%s, sb:%v, sr:%v\n", s, sb, sr)

	// output:
	/*
		ASCII byte表示时的转换:
		b:98, br:98, br:b, bs:b
		ASCII rune 字符表示时的转换:
		r:99, rb:99, rb:c, rs:c
		非 ASCII rune 字符的转换:
		rc:19990, rcb:99, rcb:c, rcs:世
		文本字符串的转换:
		s:hi, 世界, sb:[104 105 44 32 228 184 150 231 149 140], sr:[104 105 44 32 19990 30028]
	*/
}
```

以上代码中有几点需要说明：
- 当 rune 保存的字符是非 ASCII 字符时，转换成 byte 是错误的
- []rune(s) 时，会将一个字符串转换成一个码点切片，码点切片是可索引获取单个字符的

### utf8 和 unicode TODO

### string 相关文章

- [Strings, bytes, runes and characters in Go](strings_bytes_runes_characters_in_go.md)
- https://blog.golang.org/normalization