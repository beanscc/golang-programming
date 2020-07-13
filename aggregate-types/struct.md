## struct（结构体）

### 空结构体

`struct{}` 代表的是不包含任何字段的结构体类型，即空结构体类型

```go
// 声明一个结构体，空结构体
type emptyStuct struct{}
```

空结构体在创建实例时，不会分配任何内存，`并且所有该类型的变量都拥有相同的内存地址`。

这种结构适合创建没有任何状态的类型。建议用于传递 `信号` 的channel 都以 `struct{}` 作为元素类型，除非需要传递更多的信息

### 内嵌类型

## 解引用

无论使用接收者类型的值来调用这个方法，还是使用接收者类型值的指针来调用这个方法，编译器都会正确的引用或者解引用对应的值。示例如下：

```go
// 方法声明为使用 defaultMatcher 类型的值作为接收者
func (m defaultMatcher) Search(feed *Feed, searchTerm string)

// 声明一个指向 defaultMatcher 类型值的指针
dm := new(defaultMatch)

// 编译器会解开 dm 指针的引用,使用对应的值调用方法
dm.Search(feed, "test")

// 方法声明为使用指向 defaultMatcher 类型值的指针作为接收者
func (m *defaultMatcher) Search(feed *Feed, searchTerm string)

// 声明一个 defaultMatcher 类型的值
var dm defaultMatch

// 编译器会自动生成指针引用 dm 值,使用指针调用方法
dm.Search(feed, "test")
```

> **`Note:`**
> 与直接通过值或者指针调用方法不同，若通过接口（intreface）类型的值调用方法，规则有所不同。
> （1） 使用指针作为接收者声明的方法，只能在接口类型的值是一个指针的时候被调用
> （2） 使用值作为接收者声明的方法，在接口类型的值或者指针时，都可以被调用

```go

type Matcher interface {
    Search(feed *Feed, searchTerm string) ([]*Result, error)
}

// 方法声明为使用指向 defaultMatcher 类型值的指针作为接收者
func (m *defaultMatcher) Search(feed *Feed, searchTerm string)

// 通过 interface 类型的值来调用方法
var dm defaultMatcher
var matcher Matcher = dm
// 将值赋值给接口类型
matcher.Search(feed, "test") // 使用值来调用接口方法

> go build
cannot use dm (type defaultMatcher) as type Matcher in assignment


// 方法声明为使用 defaultMatcher 类型的值作为接收者
func (m defaultMatcher) Search(feed *Feed, searchTerm string)

// 通过 interface 类型的值来调用方法
var dm defaultMatcher
var matcher Matcher = &dm
// 将指针赋值给接口类型
matcher.Search(feed, "test") // 使用指针来调用接口方法

> go build
Build Successful
```