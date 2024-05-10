# 内存管理

参考文章：

- golang 内存管理分析：https://mp.weixin.qq.com/s/KGBqxJZsxzrFO0wxmYHftw
- go 内存模型：https://colobu.com/2021/07/13/Updating-the-Go-Memory-Model/
- go 内存对齐的好处：https://mp.weixin.qq.com/s/cg0pq6X1eGlm2lbD14F_bA
- 揭秘Golang内存管理优化！三色标记法源码浅析：https://mp.weixin.qq.com/s/A2aCo9UYyI3iHCu9nsGrAA
- go 内存管理：https://mp.weixin.qq.com/s/-ulE2tXl3amaQ12X-b5Xmw

## 内存分配-malloc

编程语言的内存分配器一般的分配方法：
- 线性分配器（sequential allocator, bump allocator）
- 空闲链表分配器（free-list allocator）


Go 的内存分配器是基于线程缓存分配器（thread-caching malloc, [TCMalloc](http://goog-perftools.sourceforge.net/doc/tcmalloc.html)）设计发散而来

### 分级分配


Go 的内存分配器是借鉴 TCMalloc 的设计实现高速的内存分配，其核心思想是使用多级缓存，将对象根据大小分类，并按照类别实施不同的分配策略

#### 虚拟内存布局

> Go 1.10 以前， 堆区的内存空间都是连续的；1.11 版本开始，使用稀疏的堆内存空间代替连续的内存

**线性内存**

线性内存在 C 和 Go 混合使用时会导致程序崩溃：
- 分配的内存地址会发生冲突，导致堆的初始化和扩容失败
- 没有被预留的大块内存可能会被分配给 C 语言的二进制，导致扩容后的堆不连续

**稀疏内存**



### 组件

- fixAlloc: 用于固定大小对象的空闲链表分配器，用来管理分配器使用的内存

- mheap: malloc 堆区，以 page （4096b）为粒度进行管理
- mspan: 堆管理的一系列 page
- mcental: 给定大小类型的共享空闲链表
- mcache: 小对象的线程（在 go 中，即每个 P）缓存



### 初始化



内存分配初始化

```go
func mallocinit() {
	initSizes()  // 初始化 sizeclass

    ...
    
	// Set up the allocation arena, a contiguous area of memory where
	// allocated data will be found.  The arena begins with a bitmap large
	// enough to hold 4 bits per allocated word.
	if ptrSize == 8 && (limit == 0 || limit > 1<<30) {
		// On a 64-bit machine, allocate from a single contiguous reservation.
		// 512 GB (MaxMem) should be big enough for now.
		//
		// The code will work with the reservation at any address, but ask
		// SysReserve to use 0x0000XXc000000000 if possible (XX=00...7f).
		// Allocating a 512 GB region takes away 39 bits, and the amd64
		// doesn't let us choose the top 17 bits, so that leaves the 9 bits
		// in the middle of 0x00c0 for us to choose.  Choosing 0x00c0 means
		// that the valid memory addresses will begin 0x00c0, 0x00c1, ..., 0x00df.
		// In little-endian, that's c0 00, c1 00, ..., df 00. None of those are valid
		// UTF-8 sequences, and they are otherwise as far away from
		// ff (likely a common byte) as possible.  If that fails, we try other 0xXXc0
		// addresses.  An earlier attempt to use 0x11f8 caused out of memory errors
		// on OS X during thread allocations.  0x00c0 causes conflicts with
		// AddressSanitizer which reserves all memory up to 0x0100.
		// These choices are both for debuggability and to reduce the
		// odds of a conservative garbage collector (as is still used in gccgo)
		// not collecting memory because some non-pointer block of memory
		// had a bit pattern that matched a memory address.
		//
		// Actually we reserve 544 GB (because the bitmap ends up being 32 GB)
		// but it hardly matters: e0 00 is not valid UTF-8 either.
		//
		// If this fails we fall back to the 32 bit memory mechanism
		//
		// However, on arm64, we ignore all this advice above and slam the
		// allocation at 0x40 << 32 because when using 4k pages with 3-level
		// translation buffers, the user address space is limited to 39 bits
		// On darwin/arm64, the address space is even smaller.
		arenaSize := round(_MaxMem, _PageSize) // 按页对齐, 1<<39 即 512GB
		bitmapSize = arenaSize / (ptrSize * 8 / 4)  // ptrSize = 8, bimmap 32G
		spansSize = arenaSize / _PageSize * ptrSize  // 512MB
		spansSize = round(spansSize, _PageSize) // 对齐
		for i := 0; i <= 0x7f; i++ {
			switch {
			case GOARCH == "arm64" && GOOS == "darwin":
				p = uintptr(i)<<40 | uintptrMask&(0x0013<<28)
			case GOARCH == "arm64":
				p = uintptr(i)<<40 | uintptrMask&(0x0040<<32)
			default:
				p = uintptr(i)<<40 | uintptrMask&(0x00c0<<32)
			}
			pSize = bitmapSize + spansSize + arenaSize + _PageSize
			p = uintptr(sysReserve(unsafe.Pointer(p), pSize, &reserved))
			if p != 0 {
				break
			}
		}
	}

	// PageSize can be larger than OS definition of page size,
	// so SysReserve can give us a PageSize-unaligned pointer.
	// To overcome this we ask for PageSize more and round up the pointer.
	p1 := round(p, _PageSize)

    // 初始化 spans/bitmap/arena 地址位置
	mheap_.spans = (**mspan)(unsafe.Pointer(p1))
	mheap_.bitmap = p1 + spansSize
	mheap_.arena_start = p1 + (spansSize + bitmapSize)
	mheap_.arena_used = mheap_.arena_start
	mheap_.arena_end = p + pSize
	mheap_.arena_reserved = reserved

    ...
    
	// Initialize the rest of the allocator.
	mHeap_Init(&mheap_, spansSize)
	_g_ := getg()
	_g_.m.mcache = allocmcache()
}
```



下面介绍各部分初始化情况

- 初始化对象大小尺寸

```go
/* 表字段含义说明
class: sizeclas 对象大小尺寸编号；0 表示大对象
bytes/obj: 每个对象最大占用字节大小，最小占用 1b
bytes/span: 每个 span 有多少空间, pagesize == 8192, class_to_allocnpages[sizeclass] == npage， 即 pagesize * class_to_allocnpages[sizeclass]
objects: 最大可以存储的对象个数
tail waste: 最大对象个数，当每个对象都占用最大空间时(bytes/obj 的值) 时，span 浪费的空间（即剩余未使用的空间）；如class 1的 tail waste = 8192 - (8 * 1024)
max wastte: 最大对象个数，当每个对象都占用最小空间时(1b) 时，span 浪费的空间（即剩余未使用的空间）; 如 class 1 的 max waste = (8192 - (1 * 1024)) / 8192
*/

// class  bytes/obj  bytes/span  objects  tail waste  max waste
//     1          8        8192     1024           0     87.50%
//     2         16        8192      512           0     43.75%
//     3         24        8192      341           8     29.24%
//     4         32        8192      256           0     21.88%
//     5         48        8192      170          32     31.52%
//     6         64        8192      128           0     23.44%
//     7         80        8192      102          32     19.07%
//     8         96        8192       85          32     15.95%
//     9        112        8192       73          16     13.56%
//    10        128        8192       64           0     11.72%
//    11        144        8192       56         128     11.82%
//    12        160        8192       51          32      9.73%
//    13        176        8192       46          96      9.59%
//    14        192        8192       42         128      9.25%
//    15        208        8192       39          80      8.12%
//    16        224        8192       36         128      8.15%
//    17        240        8192       34          32      6.62%
//    18        256        8192       32           0      5.86%
//    19        288        8192       28         128     12.16%
//    20        320        8192       25         192     11.80%
//    21        352        8192       23          96      9.88%
//    22        384        8192       21         128      9.51%
//    23        416        8192       19         288     10.71%
//    24        448        8192       18         128      8.37%
//    25        480        8192       17          32      6.82%
//    26        512        8192       16           0      6.05%
//    27        576        8192       14         128     12.33%
//    28        640        8192       12         512     15.48%
//    29        704        8192       11         448     13.93%
//    30        768        8192       10         512     13.94%
//    31        896        8192        9         128     15.52%
//    32       1024        8192        8           0     12.40%
//    33       1152        8192        7         128     12.41%
//    34       1280        8192        6         512     15.55%
//    35       1408       16384       11         896     14.00%
//    36       1536        8192        5         512     14.00%
//    37       1792       16384        9         256     15.57%
//    38       2048        8192        4           0     12.45%
//    39       2304       16384        7         256     12.46%
//    40       2688        8192        3         128     15.59%
//    41       3072       24576        8           0     12.47%
//    42       3200       16384        5         384      6.22%
//    43       3456       24576        7         384      8.83%
//    44       4096        8192        2           0     15.60%
//    45       4864       24576        5         256     16.65%
//    46       5376       16384        3         256     10.92%
//    47       6144       24576        4           0     12.48%
//    48       6528       32768        5         128      6.23%
//    49       6784       40960        6         256      4.36%
//    50       6912       49152        7         768      3.37%
//    51       8192        8192        1           0     15.61%
//    52       9472       57344        6         512     14.28%
//    53       9728       49152        5         512      3.64%
//    54      10240       40960        4           0      4.99%
//    55      10880       32768        3         128      6.24%
//    56      12288       24576        2           0     11.45%
//    57      13568       40960        3         256      9.99%
//    58      14336       57344        4           0      5.35%
//    59      16384       16384        1           0     12.49%
//    60      18432       73728        4           0     11.11%
//    61      19072       57344        3         128      3.57%
//    62      20480       40960        2           0      6.87%
//    63      21760       65536        3         256      6.25%
//    64      24576       24576        1           0     11.45%
//    65      27264       81920        3         128     10.00%
//    66      28672       57344        2           0      4.91%
//    67      32768       32768        1           0     12.50%
```



- 初始化 spans/bitmap/arena 地址

```go
	p1 := round(p, _PageSize)

    // 根据上面计算好的 arenaSize/spansSize/bitmapSize 初始化 spans/bitmap/arena 地址位置
	mheap_.spans = (**mspan)(unsafe.Pointer(p1))
	mheap_.bitmap = p1 + spansSize
	mheap_.arena_start = p1 + (spansSize + bitmapSize)
	mheap_.arena_used = mheap_.arena_start
	mheap_.arena_end = p + pSize
	mheap_.arena_reserved = reserved
```



- `mHeap_Init(&mheap_, spansSize)` 堆初始化

```go
// Initialize the heap.
func mHeap_Init(h *mheap, spans_size uintptr) {
	fixAlloc_Init(&h.spanalloc, unsafe.Sizeof(mspan{}), recordspan, unsafe.Pointer(h), &memstats.mspan_sys)  // 初始化 span 分配器
	fixAlloc_Init(&h.cachealloc, unsafe.Sizeof(mcache{}), nil, nil, &memstats.mcache_sys) // 初始化 mcache 分配器
	fixAlloc_Init(&h.specialfinalizeralloc, unsafe.Sizeof(specialfinalizer{}), nil, nil, &memstats.other_sys) // 初始化 specialfinalizer
	fixAlloc_Init(&h.specialprofilealloc, unsafe.Sizeof(specialprofile{}), nil, nil, &memstats.other_sys) // 初始化 specialprofilealloc

	// h->mapcache needs no init
	for i := range h.free { // 初始化每个 sizeclass 的 free / busy 数组中的双向链表 mspan
		mSpanList_Init(&h.free[i])
		mSpanList_Init(&h.busy[i])
	}

	mSpanList_Init(&h.freelarge) // 初始化 freelarge 双向链表 mspan
	mSpanList_Init(&h.busylarge) // 初始化 busylarge 双向链表 mspan
	for i := range h.central {
		mCentral_Init(&h.central[i].mcentral, int32(i)) // 初始化 每个sizeclass 的 mcentral
	}

    // 初始化 全局变量h_spans (切片 []*mspan ) ，将其指向 heap.spans 并设置长度及容量
	sp := (*slice)(unsafe.Pointer(&h_spans)) 
	sp.array = unsafe.Pointer(h.spans)
	sp.len = int(spans_size / ptrSize)
	sp.cap = int(spans_size / ptrSize)
}
```



- 初始化 `_g_.m.mcache = allocmcache()` 分配mcache, 并初始化每个 sizeclass 为 emptymspan

```go
// dummy MSpan that contains no free objects.
var emptymspan mspan

func allocmcache() *mcache {
	lock(&mheap_.lock)
	c := (*mcache)(fixAlloc_Alloc(&mheap_.cachealloc)) // 获取一个 mcache 分配器
	unlock(&mheap_.lock)
	memclr(unsafe.Pointer(c), unsafe.Sizeof(*c))  // 内存清理
	for i := 0; i < _NumSizeClasses; i++ {
		c.alloc[i] = &emptymspan // 每个 sizeclass 初始化为 emptymspan
	}

	// Set first allocation sample size.
	rate := MemProfileRate
	if rate > 0x3fffffff { // make 2*rate not overflow
		rate = 0x3fffffff
	}
	if rate != 0 {
		c.next_sample = int32(int(fastrand1()) % (2 * rate))
	}

	return c
}
```





### 分配（alloc）



TODO 总体策略：



根据对象大小不同，go 按以下规则按大小将对象分为 3 种：
- 微对象：(1, 16b) 且非指针对象
- 小对象：[16b, 32k]
- 大对象：(32k, +∞]



下面介绍不同大小对象的具体分配策略

#### 微对象



**微对象分配器**

微对象分配器将多个微对象分配请求组合到单个内存块中。当所有子对象都不可访问时，内存块才可以释放。子对象必须有 `FlagNoScan` 标识位（即没有指针），这是为了确保潜在浪费的内存数量是有限的

用于组合的内存块大小是可调的。当前设置是 16b，这与 2 倍的最坏情况内存浪费有关（当除一个子对象外的所有其他子对象都不可访问时）。8b 根本不会有浪费，但提供的组合机会更小。32b 提供的组合机会更多，但可能导致 4 倍的最坏情况内存浪费。无论块大小如何，最佳的获胜情况是 8 倍

从微对象分配器获取的对象不能显式地释放。所以当一个对象被显式地释放时，我们确保它的大小 >= maxTinySize

`SetFinalizer` 对可能来自微对象分配器的对象有一个特殊情况：它允许为内存块的内部字节设置 `finalizers`

微对象分配器的主要目标是小字符串和孤立的逃逸变量。在 json 基准测试中，分配器将分配数量减少了约 12%，堆大小减少了约 20%



#### 小对象

#### 大对象



### fixalloc



```go
type fixalloc struct {
	size   uintptr
	first  unsafe.Pointer // go func(unsafe.pointer, unsafe.pointer); f(arg, p) called first time p is returned
	arg    unsafe.Pointer
	list   *mlink
	chunk  *byte
	nchunk uint32
	inuse  uintptr // in-use bytes now
	stat   *uint64
}

// A generic linked list of blocks.  (Typically the block is bigger than sizeof(MLink).)
// Since assignments to mlink.next will result in a write barrier being preformed
// this can not be used by some of the internal GC structures. For example when
// the sweeper is placing an unmarked object on the free list it does not want the
// write barrier to be called since that could result in the object being reachable.
type mlink struct {
	next *mlink
}
```



fixalloc_Init: 初始化 f ，以分配给定大小的对象，使用分配器以获取内存块



### mspan

mspan 是一个双向链表。

```go
// mspan: runtime/mheap.go
type mspan struct {
	next     *mspan    // in a span linked list
	prev     *mspan    // in a span linked list
	start    pageID    // starting page number // span 开始的 page 号
	npages   uintptr   // number of pages in span // span 中有多少 page
	freelist gclinkptr // 空闲对象链表
	// sweep generation:
	// if sweepgen == h->sweepgen - 2, the span needs sweeping
	// if sweepgen == h->sweepgen - 1, the span is currently being swept
	// if sweepgen == h->sweepgen, the span is swept and ready to use
	// h->sweepgen is incremented by 2 after every GC

	sweepgen    uint32
	divMul      uint32   // for divide by elemsize - divMagic.mul
	ref         uint16   // capacity - number of objects in freelist
	sizeclass   uint8    // size class  // 存储对象的 sizeclass; sizeclass == 0 表示大对象
	incache     bool     // being used by an mcache
	state       uint8    // mspaninuse etc
	needzero    uint8    // needs to be zeroed before allocation
	divShift    uint8    // for divide by elemsize - divMagic.shift
	divShift2   uint8    // for divide by elemsize - divMagic.shift2
	elemsize    uintptr  // computed from sizeclass or from npages // 大对象时：s.npages << _PageShift；微/小对象时：class_to_size[sizeclass]
	unusedsince int64    // first time spotted by gc in mspanfree state // 首次在 mspanfree 状态下被 gc 发现的时间
	npreleased  uintptr  // number of pages released to the os // 已释放还给 os 的 页数
	limit       uintptr  // end of data in span
	speciallock mutex    // guards specials list
	specials    *special // linked list of special records sorted by offset.
	baseMask    uintptr  // if non-0, elemsize is a power of 2, & this will get object allocation base
}

// base 返回 mspan 的内存地址
func (s *mspan) base() uintptr {
    return uintptr(s.start << _PageShift)
}

// layout 返回 mspan 存储对象大小/可存储对象个数/占内存大小
func (s *mspan) layout() (size, n, total uintptr) {
    total = s.npages << _PageShift
    size = s.elemsize
    if size > 0 {
        n = total / size
    }
    return
}
```

### mheap

```go
type mheap struct {
	lock      mutex
	free      [_MaxMHeapList]mspan // free lists of given length // 空闲的 span 链表, 索引是 npage ，即 page 的个数
	freelarge mspan                // free lists length >= _MaxMHeapList // 空闲的大 span链表， npage > 128 的 span
	busy      [_MaxMHeapList]mspan // busy lists of large objects of given length // 在使用的 span 链表，索引时 npage
	busylarge mspan                // busy lists of large objects length >= _MaxMHeapList // 在使用中的 npage > 128 的 span
	allspans  **mspan              // all spans out there
	gcspans   **mspan              // copy of allspans referenced by gc marker or sweeper
	nspan     uint32  // span 的个数
	sweepgen  uint32 // sweep generation, see comment in mspan
	sweepdone uint32 // all spans are swept
	// span lookup
	spans        **mspan
	spans_mapped uintptr

	// Proportional sweep
	spanBytesAlloc    uint64  // bytes of spans allocated this cycle; updated atomically
	pagesSwept        uint64  // pages swept this cycle; updated atomically
	sweepPagesPerByte float64 // proportional sweep ratio; written with lock, read without

	// Malloc stats.
	largefree  uint64                  // bytes freed for large objects (>maxsmallsize)
	nlargefree uint64                  // number of frees for large objects (>maxsmallsize)
	nsmallfree [_NumSizeClasses]uint64 // number of frees for small objects (<=maxsmallsize)

	// range of addresses we might see in the heap
	bitmap         uintptr
	bitmap_mapped  uintptr
	arena_start    uintptr
	arena_used     uintptr // always mHeap_Map{Bits,Spans} before updating
	arena_end      uintptr
	arena_reserved bool

	// central free lists for small size classes.
	// the padding makes sure that the MCentrals are
	// spaced CacheLineSize bytes apart, so that each MCentral.lock
	// gets its own cache line.
	central [_NumSizeClasses]struct {
		mcentral mcentral
		pad      [_CacheLineSize]byte
	}

	spanalloc             fixalloc // allocator for span*
	cachealloc            fixalloc // allocator for mcache*
	specialfinalizeralloc fixalloc // allocator for specialfinalizer*
	specialprofilealloc   fixalloc // allocator for specialprofile*
	speciallock           mutex    // lock for special record allocators.
}
```



mHeap_SysAlloc: 扩展 heap 内存。从 os 申请新的内存扩展 heap 区（需要同步扩展 bitmap 和 spans 区，调整 arena_used 位置）



### mcache

```go
// Per-thread (in Go, per-P) cache for small objects.
// No locking needed because it is per-thread (per-P).
type mcache struct {
	// The following members are accessed on every malloc,
	// so they are grouped here for better caching.
	next_sample      int32   // trigger heap sample after allocating this many bytes
	local_cachealloc uintptr // bytes allocated from cache since last lock of heap
	local_scan       uintptr // bytes of scannable heap allocated
    
    // 微对象分配相关
	// Allocator cache for tiny objects w/o pointers.
	// See "Tiny allocator" comment in malloc.go.
	tiny             unsafe.Pointer
	tinyoffset       uintptr // 微对象分配的 offset 位置标记，是一个相对于 tiny 地址的位置的相对位移大小，每次加 tiny obj 的size
	local_tinyallocs uintptr // number of tiny allocs not counted in other stats  // 分配微对象的次数

	// The rest is not accessed on every malloc.
	alloc [_NumSizeClasses]*mspan // spans to allocate from

    // 栈内存相关
	stackcache [_NumStackOrders]stackfreelist

    // 用于本地分配器统计
	// Local allocator stats, flushed during GC.
	local_nlookup    uintptr                  // number of pointer lookups
	local_largefree  uintptr                  // bytes freed for large objects (>maxsmallsize)
	local_nlargefree uintptr                  // number of frees for large objects (>maxsmallsize)
	local_nsmallfree [_NumSizeClasses]uintptr // number of frees for small objects (<=maxsmallsize)
}
```



### mcentral



```go
// Central list of free objects of a given size.
type mcentral struct {
	lock      mutex
	sizeclass int32
	nonempty  mspan // list of spans with a free object // 有可用 obj
	empty     mspan // list of spans with no free objects (or cached in an mcache) // 无可用 obj 或者 被 mcache 借走了
}
```



### 回收

堆内存回收，以备复用

mHeap_FreeSpanLocked

mHeap_Free

mHeap_FreeStack

### 释放



### 屏障



store/store barrier





| 表头1 | 表头2 | 表头3 |
| ----- | ----- | ----- |
| r1    | r11   | r12   |
| r2    | r21   | r22   |
| r3    | r31   | r32   |
| r4    | r41   | R42   |





## 垃圾回收-GC



## 内存管理理论

参考：《垃圾回收算法手册》



常用分配算法：

- 顺序分配，也称线性分配
- 空闲链表分配：
    - 

