# slice

slice 是一种数据结构，是围绕动态数组的概念构建的，可以按需自动增长或缩小；    
slice 动态增长：是通过内置函数 append 来实现的，这个函数可以快速且高效地增长切片；    
slice 缩小：通过对切片再次切片来缩小一个切片的大小    

切片底层内存中也是连续块中分配的

### 数据结构

切片在运行时的结构由 `reflect.SliceHeader` 表示

```go
// runtime/slice.go
// SliceHeader 是 slice 在 runtime 的表现形式。
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}

// sliceHeader is a safe version of SliceHeader used within this package.
type sliceHeader struct {
	Data unsafe.Pointer
	Len  int
	Cap  int
}
```

其中 `Data` 字段是指向底层数组的指针， `Len` 是切片可访问元素的个数（即长度）， `Cap` 是切片允许增长到的元素个数（即容量，也是 `Data`
数组的大小）

**示例代码 slice-1**
```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	// make 创建切片
	slice := make([]int, 0, 2)
	fmt.Printf("slice: %#v, len: %v, cap:%v, data struct:%#v\n", slice, len(slice), cap(slice), (*reflect.SliceHeader)(unsafe.Pointer(&slice)))
	// output: slice: []int{}, len: 0, cap:2, data struct:&reflect.SliceHeader{Data:0xc000098010, Len:0, Cap:2}
}
```

### 创建切片

切片的创建有以下几种方式：
- 直接声明：
    - nil 切片：`var slice []int`
    - 声明时赋值：`var slice []int = []int{1, 2}`
- 使用 make 关键字: `slice := make([]int, 0, 10)` 语法 `make([]T, len, cap)` 或 `make([]T, len)` 若不指定 cap 时， cap = len； 另外其中 len <= cap，否则编译不通过
- 使用切片字面量: `slice := []int{1, 2}`
- 从其他数组和切片截取: `slice := array[1:]` 或 `slice1 := slice[1:]`

使用字面量的方式创建切片时，大部分工作会在编译期间完成，但使用 make 关键字创建切片时，很多工作需要在运行时完成

直接声明的方式如上面例子所示比较简单，下面介绍其他几种方式

#### make

查看`示例代码 slice-1` 的汇编信息: `go tool compile -S -N main.go`, 得到如下：

```
...
"".main STEXT size=1185 args=0x0 locals=0x1d8
        ...
        0x006c 00108 (main.go:11)       CALL    runtime.makeslice(SB)
        ...
```

这里我们重点关注对第 11 行的操作，可以看到 `make` 关键字创建切片时，调用了 `runtime.makeslice` 方法  

下面我们看一下运行时期间是如何通过 `runtime.makeslice` 创建切片的

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```

`makeslice` 主要是计算切片所需的内存空间，并在堆上申请一片连续的内存，用于保存切片元素

```
内存空间 = 切片元素内存占用大小 * 切片容量
```

以下情况会触发运行时panic：
- 所需内存空间大小发生了溢出
- 申请内存空间大小大于最大可分配的内存
- 设置的长度小于 0 或 长度大于容量

#### 字面量

通过字面量方式创建并初始化切片的方式比较简单
> 需要注意初始化时是否有使用索引号，因为使用索引号初始化切片时，其长度和容量并非看起来那么简单哦

```go
package main

import "fmt"

func main() {
	slice := []int{1, 3}
	fmt.Printf("slice:%#v, len:%v, cap:%v\n", slice, len(slice), cap(slice))

	slice1 := []int{0, 3: 3} // 未指定索引号的位置将会被初始化为零值
	fmt.Printf("slice1:%#v, len:%v, cap:%v\n", slice1, len(slice1), cap(slice1))

	// output:
	/*
		slice:[]int{1, 3}, len:2, cap:2
		slice1:[]int{0, 0, 0, 3}, len:4, cap:4
	*/
}
```

#### 下标截取

TODO

**切片创建示例：**

```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	// 直接声明且仅声明不赋值，此时 slice 是一个 nil 切片
	var slice []int
	fmt.Printf("nil slice: %#v, len: %v, cap:%v, data struct:%#v\n", slice, len(slice), cap(slice), (*reflect.SliceHeader)(unsafe.Pointer(&slice)))

	// 直接声明并赋值
	var slice1 []int = []int{1, 2}
	fmt.Printf("slice1: %#v, len: %v, cap:%v, data struct:%#v\n", slice1, len(slice1), cap(slice1), (*reflect.SliceHeader)(unsafe.Pointer(&slice1)))

	// 字面量创建
	slice2 := []int{2, 4}  // len 和 cap 相等
	slice21 := []int{4: 4} // len:1, cap = 5
	fmt.Printf("slice2: %#v, len: %v, cap:%v, data struct:%#v\n", slice2, len(slice2), cap(slice2), (*reflect.SliceHeader)(unsafe.Pointer(&slice2)))
	fmt.Printf("slice21: %#v, len: %v, cap:%v, data struct:%#v\n", slice21, len(slice21), cap(slice21), (*reflect.SliceHeader)(unsafe.Pointer(&slice21)))

	// make 创建时后 2 个参数依次分别指定了切片的长度len和容量cap, cap 不是必须的，若不指定 cap, 则 cap = len
	// make 创建，将根据指定的长度初始化切片，就是说切片中将包含 len 个切片元素类型的零值
	slice3 := make([]int, 2)
	fmt.Printf("slice3: %#v, len: %v, cap:%v, data struct:%#v\n", slice3, len(slice3), cap(slice3), (*reflect.SliceHeader)(unsafe.Pointer(&slice3)))

	// 容量为0的空切片
	slice4 := make([]int, 0)  // 空切片
	slice41 := make([]int, 0) // 空切片
	fmt.Printf("empty slice without cap. slice4: %#v, len: %v, cap:%v, data struct:%#v\n", slice4, len(slice4), cap(slice4), (*reflect.SliceHeader)(unsafe.Pointer(&slice4)))
	fmt.Printf("empty slice without cap. slice41: %#v, len: %v, cap:%v, data struct:%#v\n", slice41, len(slice41), cap(slice41), (*reflect.SliceHeader)(unsafe.Pointer(&slice41)))
	slice41 = append(slice41, 1)
	fmt.Printf("empty slice without cap afeter append. slice41: %#v, len: %v, cap:%v, data struct:%#v\n", slice41, len(slice41), cap(slice41), (*reflect.SliceHeader)(unsafe.Pointer(&slice41)))

	// 容量不为 0 的空切片
	slice5 := make([]int, 0, 2)  // 空切片
	slice51 := make([]int, 0, 2) // 空切片
	fmt.Printf("empty slice with cap. slice5: %#v, len: %v, cap:%v, data struct:%#v\n", slice5, len(slice5), cap(slice5), (*reflect.SliceHeader)(unsafe.Pointer(&slice5)))
	fmt.Printf("empty slice with cap. slice51: %#v, len: %v, cap:%v, data struct:%#v\n", slice51, len(slice51), cap(slice51), (*reflect.SliceHeader)(unsafe.Pointer(&slice51)))

	// output:
	/*
		nil slice: []int(nil), len: 0, cap:0, data struct:&reflect.SliceHeader{Data:0x0, Len:0, Cap:0}
		slice1: []int{1, 2}, len: 2, cap:2, data struct:&reflect.SliceHeader{Data:0xc000098050, Len:2, Cap:2}
		slice2: []int{2, 4}, len: 2, cap:2, data struct:&reflect.SliceHeader{Data:0xc0000980b0, Len:2, Cap:2}
		slice21: []int{0, 0, 0, 0, 4}, len: 5, cap:5, data struct:&reflect.SliceHeader{Data:0xc000090030, Len:5, Cap:5}
		slice3: []int{0, 0}, len: 2, cap:2, data struct:&reflect.SliceHeader{Data:0xc000098170, Len:2, Cap:2}
		empty slice without cap. slice4: []int{}, len: 0, cap:0, data struct:&reflect.SliceHeader{Data:0x1196ab8, Len:0, Cap:0}
		empty slice without cap. slice41: []int{}, len: 0, cap:0, data struct:&reflect.SliceHeader{Data:0x1196ab8, Len:0, Cap:0}
		empty slice without cap afeter append. slice41: []int{1}, len: 1, cap:1, data struct:&reflect.SliceHeader{Data:0xc000098230, Len:1, Cap:1}
		empty slice with cap. slice5: []int{}, len: 0, cap:2, data struct:&reflect.SliceHeader{Data:0xc000098280, Len:0, Cap:2}
		empty slice with cap. slice51: []int{}, len: 0, cap:2, data struct:&reflect.SliceHeader{Data:0xc000098290, Len:0, Cap:2}
	*/
}
```

#### nil 切片和空切片

注意 `nil切片` 和 `空切片`的区别：
- nil 切片中指向底层数组的指针地址为0，len为0，cap也为0
- empty 切片：
    - cap = 0 的切片，底层数组指向了同一个空数组的地址，等到空切片中追加元素时，会创建一个新的数组
    - cap > 0 的切片，底层数组已创建并初始化好了

示例中 `reflect.SliceHeader` 中 Data 所表示的内存地址，会在切片触发扩容/所容后改变，并非一成不变的

#### 访问

- 单个元素访问：直接通过索引访问切片索引位置的元素
- 迭代遍历：`for i:=0; i<len(slice); ++` 或 `for i := range slice`  或 `for i, v := range slice` 

#### copy

TODO 

copy 

#### append 追加

// runtime.growslice TODO 分析


### 在函数间传递切片

在函数间传递切片，就是要以值的方式传递切片。由于切片的尺寸很小，在函数间复制和传递切片成本也很低。

在 64 位架构的机器上，一个切片需要 24 字节的内存:指针字段需要 8 字节，长度和容量字段分别需要 8 字节。由于与切片关联的数据包含在底层数组
里，不属于切片本身，所以将切片 复制到任意函数的时候，对底层数组大小都不会有影响。复制时只会复制切片本身，不会涉及底层数组。

在函数间传递 24 字节的数据会非常快速、简单。这也是切片效率高的地方。不需要传递指
针和处理复杂的语法，只需要复制切片，按想要的方式修改数据，然后传递回一份新的切片副本

## 相关文章

- https://blog.golang.org/slices
- https://blog.golang.org/go-slices-usage-and-internals

参考资料：
- [面向信仰编程-go-slice](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice)
- [深度解密Go语言之Slice](https://blog.csdn.net/qcrao/article/details/89506670)