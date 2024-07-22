## 布尔类型（bool）

Go 预定义的布尔类型是 `bool`，由预先声明的常量 `ture` 和 `false` 表示；是一个已定义类型

长度 1 字节



```go
// bool is the set of boolean values, true and false.
type bool bool

// true and false are the two untyped boolean values.
const (
	true  = 0 == 0 // Untyped bool.
	false = 0 != 0 // Untyped bool.
)
```

