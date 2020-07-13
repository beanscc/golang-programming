## map

go 的 map 就是 hash table. 数据存放在桶中的数组中，每个桶最多可以存放 8 对键值对。哈希值的低位用来选择桶。每个桶都包含一小部分哈希值的
高位用于在所有桶中区分一个桶

若超过 8 个键值对hash到一个桶上，则启用额外的桶

当hash 表扩容时，我们给桶的数组分配一个 2 倍的新数组。桶将被逐渐的从旧的桶数组中复制到新数组中

Map 迭代通过桶的数组，并按顺序返回keys

### 数据结构

```go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // map 中元素的数量
	flags     uint8
	B         uint8  // bucket 的数量，len(buckts) = 2^B, 桶的最大数量是：loadFactor * 2^B
	noverflow uint16 // 溢出桶数量的近似值，详细见 hmap.incrnoverflow() 方法
	hash0     uint32 // 哈希种子，能为哈希函数的结果引入不确定性，值在创建哈希表时确定，并在调用哈希函数时作为参数传入

	buckets    unsafe.Pointer // 2^B 个桶的数组，若 count==0，则为nil
	oldbuckets unsafe.Pointer // 扩容时用于保存 buckets 的字段，扩容时不会是 nil，大小时 buckets 的一半
	nevacuate  uintptr        // 稀疏进度计数器（） progress counter for evacuation (buckets 小于改值时已被稀疏)

	extra *mapextra // optional fields
}
```


bucket 桶的结构
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


溢出桶
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
	nextOverflow *bmap}

```

### 初始化

初始化的 map 的两种方法：
- 通过字面量初始化
- 通过 make 关键字在运行时初始化

#### 字面量

字面量初始化 map：

```go
var hash = map[string]bool{
	"int":   true,
	"uint":  true,
	"float": true,
}
```

#### 运行时 make


## 相关文章

- https://blog.golang.org/go-maps-in-action