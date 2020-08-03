# map

map 用于存储一系列无序的键值对, *迭代遍历时返回元素的顺序是乱序的*

## 数据结构

先看一下 map 运行时的相关数据结构

```go
// map flags
iterator     = 1 // there may be an iterator using buckets
oldIterator  = 2 // there may be an iterator using oldbuckets
hashWriting  = 4 // a goroutine is writing to the map
sameSizeGrow = 8 // the current map growth is to a new map of the same size


// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // map 中元素的数量
	flags     uint8 // map 当前状态标识，见 flags
	B         uint8  // 用于计算 bucket 的数量，len(buckts) = 2^B, 桶的最大数量是：loadFactor * 2^B
	noverflow uint16 // 溢出桶数量的近似值，详细见 hmap.incrnoverflow() 方法
	hash0     uint32 // 哈希种子，能为哈希函数的结果引入不确定性，值在创建哈希表时确定，并在调用哈希函数时作为参数传入

	buckets    unsafe.Pointer // *[2^B]*bmap 长度为 2^B 的桶数组，若 count==0，则为nil
	oldbuckets unsafe.Pointer // 扩容时用于保存 buckets 的字段，扩容时不会是 nil，大小是 buckets 的一半
	nevacuate  uintptr        // 稀疏进度计数器 progress counter for evacuation (buckets 小于改值时已被稀疏)

	extra *mapextra // optional fields
}
```

```go
// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8 // 8 个键值对
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

`bmap` 在运行时的字段还有更多的字段信息，由于哈希表中可存储不同类型的键值对，且 Go 不支持范型，所以键值对占用的内存大小只能在编译期间进行推导；这些隐含字段在运行时也是通过计算内存地址的方式直接访问的，根据编译期间的 `cmd/compileinternal/gc.bmap` 函数对它的结构重建：

```go
type bmap struct {
	topbits  [8]uint8
	keys     [8]keytype
	elems    [8]elemtype
	pad      uintptr
	overflow uintptr
}
```

```go
// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}
```

map 其实是一个 hash table. 数据保存在桶（`bucket`）中的数组中，每个桶最多可以存放 8 对键值对。hash 冲突采用拉链法解决；若 hash 选中的 bucket数组位置已满，则创建溢出桶继续保存数据。key 的哈希值的*低位*用来选择 key 存放在哪个 bucket, *高 8 位*用来确定 key 在 bucket 中数组的哪个索引位置

当 hash 表扩容时，bucket 的数量是以 2^B 增长。将逐渐的从旧桶中复制数据到新桶

## 初始化

初始化的 map 的两种方法：
- 通过字面量初始化
- 通过 make 关键字在运行时初始化


字面量初始化 map：

```go
package main

import (
	"fmt"
)

func main() {
	m := map[string]int{
		"1": 1,
	}
	fmt.Println(m)
}
```

使用 `make` 关键字初始化：

```go
package main

import (
	"fmt"
)

func main() {
	m := make(map[string]int, 2)
	m["1"] = 1
	fmt.Println(m)
}
```

以上 2 种初始化方式，无论通过哪种方式，我们通过汇编查看发现，都会调用 `runtime.makemap_small` 来初始化 map，然后调用 `runtime.mapassign_faststr` 来进行赋值

`makemap_small` 其实是 `makemap` 的同类函数之一，还有同类函数`makemap64`, 这里主要看一下 `makemap`:

```go
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```


## 查找定位 key

v := map[k] 和 v, ok := map[k] 两种操作如何实现的？

## 赋值

## 删除

## 扩容和迁移


## 迭代遍历

## 相关文章

- https://blog.golang.org/go-maps-in-action


## 参考资料

- go-in-action