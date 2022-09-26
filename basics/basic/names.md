## 命名（names）



Go 语言中的函数名、变量名、常量名、类型名、语句标签名（带标签的语句可能是 `goto`、`break` 或 `continue` 语句的目标语句）还有包名，都遵循一个简单的命名规则：

- 必须以一个字母（unicode 字母）或下划线开始，后面可以跟任意数量的字母、数字和下划线

> 命名是大小写敏感的：heapSort 和 HeapSort 是不同的标识符实体



Go 拥有 25 个保留关键字，关键字不能作为标识符名，只能在特定的语法结构中使用。以下是关键字：

```
break        default      func         interface    select
case         defer        go           map          struct
chan         else         goto         package      switch
const        fallthrough  if           range        type
continue     for          import       return       var
```

### 预定义标识符

此外，还有以下几十个 **预定义标识符** ，比如 `int` 和 `true` 等用于内建常量、类型和函数

```
Types:
	bool byte complex64 complex128 error float32 float64
	int int8 int16 int32 int64 rune string
	uint uint8 uint16 uint32 uint64 uintptr

Constants:
	true false iota

Zero value:
	nil

Functions:
	append cap close complex copy delete imag len
	make new panic print println real recover
```

以上这些预定义标识符不是保留关键字，所以可以将其作为标识符名



### 包命名

参见[包命名规则](package-names.md)
