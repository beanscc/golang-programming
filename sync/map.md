## sync.Map

并发安全的 map

首先，看下 Map 的内部结构：

```go
// Map 如同 Go 的 map[interface{}]interface{}, 但是在多个 goroutines 中没有额外锁的情况下使用是并发安全的
// 读取, 存储, 和删除方法的时间复杂度平均是常量
//
// Map 类型是专用的。大部分代码应该使用带有单独锁定和协调功能的普通 Go map，以提升类型安全性并使其更容易在 map 内容间维护其他不变性
//
// Map 类型针对 2 种常用用例进行了优化：
//  (1) 一个给定 key 只写一次，但会多次读取，类似只增缓存
//  (2) 多个 goroutine 对不相交的 key，进行读，写，并覆写时
// 以上 2 种用例，相较于 Go map 搭配单独的 Mutex 或 RWMutex 而言，使用 Map 可能显著的减少锁竞争
// 
// 零值 Map 为空，并且可以立即使用. 使用过的 Map 不能复制
type Map struct {
	mu Mutex

    // read 包含部分并发访问安全的（使用或不使用 mu）map 的内容，
    //
    // read 字段本身总是读取安全的，但是存储时必须使用 mu
	//
	// 存储在 read 中的 Entries 可能在不使用 mu 的情况下被更新，但是更新一个之前已清除的 entry，
	// 需要将 entry 被复制到 dirty map 中并使用 mu 标记为未清除的
	read atomic.Value // readOnly

    // dirty 包含部分需要使用 mu 的 map 内容。为了确保 dirty map 中数据可以被尽快提升到 read map 中， 
    // dirty 中也包含了 read 中所有未清除的 entries
	//
    // 已清除的 entries 不会被保存到 dirty 中。clean map 中一个已清除的 entry，在保存新值前，必须被标记成未清除的，
    // 并添加到 dirty 中
    //
    // 若 dirty map 为 nil, 下一个写入 dirty map 时将通过浅复制（忽略已标记清除项） clean map 来初始化它 
	dirty map[interface{}]*entry

    // misses 统计自从 read map 最后一次更新后的 load 次数，需要锁定 mu 以判定 key 是否存在
	//
    // 一旦 misses 的值足够涵盖复制 dirty map 的成本，dirty map 将被提升到 read map（以未修正的状态），
    // 并且下次存储时，将再次创建一个新的 dirty 副本
	misses int
}
```