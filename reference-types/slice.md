# slice

slice 是一种数据结构，是围绕动态数组的概念构建的，可以按需自动增长或缩小；    
slice 动态增长：是通过内置函数 append 来实现的，这个函数可以快速且高效地增长切片；    
slice 缩小：通过对切片再次切片来缩小一个切片的大小    

切片底层内存中也是连续块中分配的

## 数据结构

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

## 创建和初始化

切片的创建有以下几种方式：
- 直接声明：
    - nil 切片：`var slice []int`
    - 声明时赋值：`var slice []int = []int{1, 2}`
- 使用 make 关键字: `slice := make([]int, 0, 10)` 语法 `make([]T, len, cap)` 或 `make([]T, len)` 若不指定 cap 时， cap = len； 另外其中 len <= cap，否则编译不通过
- 使用切片字面量: `slice := []int{1, 2}`
- 从其他数组和切片截取: `slice := array[1:]` 或 `slice1 := slice[1:]`

使用字面量的方式创建切片时，大部分工作会在编译期间完成，但使用 make 关键字创建切片时，很多工作需要在运行时完成

直接声明的方式如上面例子所示比较简单，下面介绍其他几种方式

### make

使用 `make` 关键字声明并初始化切片的语法：

```go
// 使用长度声明
slice := make([]T, len)  // 只指定长度是，其容量==长度

// 使用长度和容量声明
slice := make([]T, len, cap) // cap >= len && len >=0; 其底层数组的长度即为此处的 cap
``` 

`make` 命令是如何工作的？

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

### 字面量

通过字面量方式创建并初始化切片的方式比较简单
> 需要注意初始化时是否有使用索引声明切片，因为使用索引声明切片时，其长度和容量并非看起来那么简单哦

字面量声明切片有以下几种方式：
- 字面量声明切片：`slice := []int{1, 2}` 也可以声明一个空切片 `slice := []int{}`
- 字面量使用索引声明切片：`slice := []int{0, 3:3}`，此时得到的切片 `slice: []int{0, 0, 0, 3}, len:4, cap:4`

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

### nil 切片和空切片

注意 `nil切片` 和 `空切片`的区别：
- nil 切片：指向底层数组的指针地址为0，len为0，cap也为0
- empty 切片：底层数组包含 0 个元素，也没有分配任何存储空间，len为0，cap也为0


创建 nil 切片：`var slice []int`
创建空切片：
    - 使用 `make`: `slice := make([]int, 0)`
    - 使用字面量: `slice := []int{}`

### 截取数组或切片

通过截取数组或者切片的一部分可以创建新的切片，**得到的新切片和原始数组或切片共享同一段底层数组**

`slice := []int{1, 3, 5, 7, 9}` 有以下截取方式：
- 2 个索引: `slice1 := slice[start:end]`
	- `slice1 := slice[:]`
	- `slice1 := slice[1:]`
	- `slice := slice[:3]`
	- `slice1 := slice[1:3]`
- 3 个索引: `slice1 := slice[1:3:3]` (`slice1 := slice[start:end:cap]`)

若切片slice的容量为 m，则 slice[i:j] 其 len: j - i; cap = m - i； 则 slice[i:j:k] 其 len: j - i; cap: k - i

因为截取后得到得新切片和原切片共享同一段底层数组，所以若这段共享数组的任何一方索引位置值的改变都将同时改变另一方相应索引位置的相同变化

示例：
```go
package main

import (
	"fmt"
)

func main() {
	slice := []int{1, 3, 5, 7, 9}
	// 通过截取已有切片，创建一个新的切片; 其长度为2，容量为4
	slice1 := slice[1:3]
	fmt.Printf("slice1:%#v, len:%v, cap:%v\n", slice1, len(slice1), cap(slice1))

	slice1[0] = 2 // slice[1] 也相应改变为 2
	fmt.Printf("slice:%#v, slice:%#v\n", slice, slice1)

	// output:
	/*
		slice1:[]int{3, 5}, len:2, cap:4
		slice:[]int{1, 2, 5, 7, 9}, slice:[]int{2, 5}
	*/
}
```

### 切片创建初始化示例

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

	// 空切片
	slice4 := make([]int, 0)  // 空切片
	slice41 := make([]int, 0) // 空切片
	fmt.Printf("empty slice without cap. slice4: %#v, len: %v, cap:%v, data struct:%#v\n", slice4, len(slice4), cap(slice4), (*reflect.SliceHeader)(unsafe.Pointer(&slice4)))
	fmt.Printf("empty slice without cap. slice41: %#v, len: %v, cap:%v, data struct:%#v\n", slice41, len(slice41), cap(slice41), (*reflect.SliceHeader)(unsafe.Pointer(&slice41)))
	slice41 = append(slice41, 1)
	fmt.Printf("empty slice without cap after append. slice41: %#v, len: %v, cap:%v, data struct:%#v\n", slice41, len(slice41), cap(slice41), (*reflect.SliceHeader)(unsafe.Pointer(&slice41)))

	// 指定容量，但长度为 0 的切片
	slice5 := make([]int, 0, 2)
	slice51 := make([]int, 0, 2)
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
		empty slice without cap after append. slice41: []int{1}, len: 1, cap:1, data struct:&reflect.SliceHeader{Data:0xc000098230, Len:1, Cap:1}
		empty slice with cap. slice5: []int{}, len: 0, cap:2, data struct:&reflect.SliceHeader{Data:0xc000098280, Len:0, Cap:2}
		empty slice with cap. slice51: []int{}, len: 0, cap:2, data struct:&reflect.SliceHeader{Data:0xc000098290, Len:0, Cap:2}
	*/
}
```

## 使用

### 访问和赋值

- 元素访问：通过索引访问切片索引位置的元素
- 赋值：通过索引改变索引位置的值
- 迭代遍历：`for i:=0; i<len(slice); i++` 或 `for i := range slice`  或 `for i, v := range slice` 


通过 `for range` 关键字迭代切片时，`i` 表示当前元素在切片中的索引位置，`v` 表示当前元素的**副本**


```go
package main

import "fmt"

func main() {
	slice := []int{1, 3}
	// 访问
	fmt.Println(slice[0])

	// 赋值
	slice[0] = 10
	fmt.Println(slice[0])

	fmt.Println("for 迭代:")
	for i := 0; i < len(slice); i++ {
		fmt.Printf("i:%d, v:%d\n", i, slice[i])
	}

	fmt.Println("for i := range 迭代:")
	for i := range slice {
		fmt.Printf("i:%d, v:%d\n", i, slice[i])
	}

	fmt.Println("for i, v := range 迭代:")
	for i, v := range slice {
		fmt.Printf("i:%d, v:%d\n", i, v)
	}

	// output:
	/*
		1
		10
		for 迭代:
		i:0, v:10
		i:1, v:3
		for i := range 迭代:
		i:0, v:10
		i:1, v:3
		for i, v := range 迭代:
		i:0, v:10
		i:1, v:3
	*/
}
```

### copy

```go
// 内置函数 从 src 切片中复制元素到 dst 切片中（一个特殊例子，也可以从 string 中复制字节到字节切片中）
// src 和 dst 可能重叠。copy 函数返回复制的元素数量，这个数将是 len(src) 和 len(dst) 中最小的那个
func copy(dst, src []Type) int
```

copy 如其名，就是 copy

```go
package main

import (
	"fmt"
)

func main() {
	slice := []byte(`hello, world`)
	buf := make([]byte, 5)
	copy(buf, slice)
	buf[0] = 'H'
	fmt.Println(string(slice))
	fmt.Println(string(buf))
	// output:
	/*
		hello, world
		Hello
	*/
}
```

### append 追加

append() 用于向切片中追加元素，根据函数定义，一次可追加一个或多个，下面是 append() 函数原型

```go
// append 内置函数用于向切片尾部追加元素
// 若切片有足够的容量，目标切片将保存追加的元素到底层数组，并返回新的切片
// 若切片底层数组容量不足以容纳新追加的元素，底层将分配一个新的更大容量的数组，并复制原数组元素到新数组，然后
// 将追加元素保存到新数组中，然后然后更新后的切片。因此有必要保存 append 的结果到 slice 变量本身：
//   slice = append(slice, e1, e2)
//   slice = append(slice, anotherSlice...)
// 作为一个特殊的例子，append 一个 string 到 []byte 是合法的，如：
//   slice = append([]byte("hello "), "world"...)
func append(slice []Type, elems ...Type) []Type
```

append 向切片中追加元素，其实是向切片底层数组中添加元素，但底层数组的长度是固定的，一旦定义后，就不可修改。当切片索引指向的底层数组
的最后一个元素已保存有值，就无法继续向该数组中追加元素了。这时，slice 的 append 操作就会申请一块更大内存的用于容纳原底层数组所有元素和新追加
元素。为了避免每次 append 都触发底层数组的扩容，会在每次扩容时候预留一定空间，每次扩容都需要迁移所有数据，成本较大

下面看一下 `append()` 时底层数组长度不够时，需要扩容时的情况；

```go
package main

import (
	"fmt"
)

func main() {
	slice := []int{1, 2}
	slice = append(slice, 3)
	fmt.Println(slice) // output: [1 2 3]
}
```

查看以上代码的汇编，查看 append 的实现（`go tool compile -S -N main.go`）

```bash
...
"".main STEXT size=645 args=0x0 locals=0x108
	...
	0x00ac 00172 (main.go:9)	CALL	runtime.growslice(SB)
	...
```

可以看到 `append()` 其实是通过 `runtime.growslice` 函数实现的

// runtime.slice.go
```go
// growslice handles slice growth during append.
// It is passed the slice element type, the old slice, and the desired new minimum capacity,
// and it returns a new slice with at least that capacity, with the old data
// copied into it.
// The new slice's length is set to the old slice's length,
// NOT to the new requested capacity.
// This is for codegen convenience. The old slice's length is used immediately
// to calculate where to write new values during an append.
// TODO: When the old backend is gone, reconsider this decision.
// The SSA backend might prefer the new length or to return only ptr/cap and save stack space.
func growslice(et *_type, old slice, cap int) slice {
	// 计算扩容后的容量 cap
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	// lenmem: 原切片len所需的内存大小
	// newlenmem: 新切片追加元素后长度的内存大小
	// capmem: 新容量所需内存对齐后的内存大小
	// newcap: 内存对齐后重新计算的新容量（可能比之前的容量稍大一些）
	// overflow: 扩容所需内存是否会溢出
	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```

`runtime.roundupsize` 函数根据需要的内存大小，进行内存分配对齐后，返回内存分配器分配的内存大小（这个值可能大于传入的所需内存大小）；
这里涉及到内存分配相关

// runtime.msize.go
```go
// Returns size of the memory block that mallocgc will allocate if you ask for the size.
func roundupsize(size uintptr) uintptr {
	if size < _MaxSmallSize { // 小对象的最大内存大小
		if size <= smallSizeMax-8 { // smallSizeMax 小对象的最小内存大小
			return uintptr(class_to_size[size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]])
		} else {
			return uintptr(class_to_size[size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]])
		}
	}
	if size+_PageSize < size {
		return size
	}
	return round(size, _PageSize)
}
```

**当 slice 切片底层数组还有容量时， append(slice, v) 操作后 slice 发生了什么变化？**

看下面这段代码：
```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	slice := make([]int, 1, 10)
	fmt.Println("slice before append:", slice)
	slice1 := append(slice, 10)
	fmt.Println("slice after append:", slice)
	fmt.Println("slice1:", slice1)
	// output
	/*
		slice before append: [0]
		slice after append: [0]
		slice1: [0 10]
	*/
}
```

其中 `slice` 切片底层数组有足够的容量，可以容纳 `10`，但在 `append` 操作后，打印 `slice` 结果同之前一样无任何变化。真的如此吗？
我们来看下底层的数据的变化：

```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	slice := make([]int, 1, 10)
	fmt.Println("slice before append:", slice, (*reflect.SliceHeader)(unsafe.Pointer(&slice)))

	slice1 := append(slice, 10)
	fmt.Println("slice after append:", slice, (*reflect.SliceHeader)(unsafe.Pointer(&slice)))
	fmt.Println("slice1:", slice1, (*reflect.SliceHeader)(unsafe.Pointer(&slice1)))

	v := *(*[10]int)(unsafe.Pointer((*reflect.SliceHeader)(unsafe.Pointer(&slice)).Data)) // slice 底层 array
	fmt.Println("the underlying array of slice:", v)

	// output
	/*
		slice before append: [0] &{824633835760 1 10}
		slice after append: [0] &{824633835760 1 10}
		slice1: [0 10] &{824633835760 2 10}
		the underlying array of slice: [0 10 0 0 0 0 0 0 0 0]
	*/
}
```

可以看到 `append()` 前后 `slice` 的结构看上去没有变化，但我们通过 `unsafe.Pointer` 访问其底层数组时，可以看出其实底层数组已经有了变化，
已经将 `10` 追加到了数组中，只是由于 `append()` 操作并为改变传入 `slice` 的 `len` 和 `cap` 字段值，所以虽然 `10` 已存入底层数组 `arr[1]` 
索引位置，但 `len(slice) == 1`，索引它无法访问底层数组索引为 1 的元素值，仅仅能访问索引为 0 的元素值 0


## 在函数间传递切片

在函数间传递切片，就是要以值的方式传递切片。由于切片的尺寸很小，在函数间复制和传递切片成本也很低。

在 64 位架构的机器上，一个切片需要 24 字节的内存:指针字段需要 8 字节，长度和容量字段分别需要 8 字节。由于与切片关联的数据包含在底层数组
里，不属于切片本身，所以将切片 复制到任意函数的时候，对底层数组大小都不会有影响。复制时只会复制切片本身，不会涉及底层数组。

在函数间传递 24 字节的数据会非常快速、简单。这也是切片效率高的地方。不需要传递指针和处理复杂的语法，只需要复制切片，按想要的方式修改数据,
然后传递回一份新的切片副本

## 相关文章

- [Arrays, slices (and strings): The mechanics of 'append'](https://blog.golang.org/slices)
- [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)

参考资料：
- [面向信仰编程-go-slice](https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-array-and-slice)
- [深度解密Go语言之Slice](https://blog.csdn.net/qcrao/article/details/89506670)