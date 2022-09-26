# sync

同步

## 锁

### 互斥锁

`sync.Mutex` 类型至有两个公开的指针方法

- Lock
- Unlock

> 1. 若锁定一个已锁定的互斥锁，那么进行重复锁定的操作的 goroutine 将被阻塞，直到该互斥锁回到解锁状态
> 2. 当对一个未锁定的互斥锁进行解锁操作时，就会引发一个运行时恐慌（panic），避免这种情况发生的最简单、有效的方式是使用 defer语句，更容易保证解锁操作的唯一性

### 读写锁

针对读写操作的互斥锁。

与普通的互斥锁不同之处：    
（1） 可以分别针对读操作和写操作进行锁定和解锁操作   
（2） 遵循的访问控制规则不同，读写锁控制下的多个写操作之间是互斥的，并且写操作与读操作之间也是互斥的。但是，多个读操作之间却不存在互斥关系

> 在这样的互斥策略下，读写锁可以在大大降低因使用锁而造成的性能损耗的情况下，完成对共享资源的访问控制

读写锁结构体 `sync.RWMutex`

#### 写锁

写锁定，如下：

```go
func (rw *RWMutex) Lock()
```

写解锁，如下

```go
func (rw *RWMutex) Unlock()
```

#### 读锁

读锁定，如下：

```go
func (rw *RWMutex) RLock()
```

读解锁，如下：

```go
func (rw *RWMutex) RUnlock()
```

#### RLocker

```go
func (rw *RWMutex) RLocker() * Locker {}
```

`RLocker` 该方法返回一个实现里 sync.Locker 接口类型的值。

sync.Locker 接口定义：

```go
// A Locker represents an object that can be locked and unlocked.
type Locker interface {
	Lock()
	Unlock()
}
```

实现 Locker 接口类型的实现类型

- *sync.Mutex
- *sync.RWMutex

## 条件变量

`sync.Cond` 类型代表了`条件变量`

### New Cond

```go
// NewCond returns a new Cond with Locker l.
func NewCond(l Locker) *Cond {
	return &Cond{L: l}
}
```

### *Cond.Wait()

`Wait` 方法会自动地对该条件变量关联的那个锁进行解锁，并且使它所在的 goroutine 阻塞。一旦接收到通知，该方法所在的 goroutine 就会被唤醒，并且该方法会立即尝试锁定该锁。

```go
// Wait atomically unlocks c.L and suspends execution
// of the calling goroutine. After later resuming execution,
// Wait locks c.L before returning. Unlike in other systems,
// Wait cannot return unless awoken by Broadcast or Signal.
//
// Because c.L is not locked when Wait first resumes, the caller
// typically cannot assume that the condition is true when
// Wait returns. Instead, the caller should Wait in a loop:
//
//    c.L.Lock()
//    for !condition() {
//        c.Wait()
//    }
//    ... make use of condition ...
//    c.L.Unlock()
//
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}
```

## 原子操作

原子操作即执行过程中不能被中断的操作。在针对某个值的原子操作执行过程当中，CPU绝不会再去执行其他针对该值的操作，无论这些其他操作是否为原子操作。

sync/atomic 包中函数，可以对几种简单类型的值执行原子操作。这些类型包括以下 6 种：
- int32
- int64
- uint32
- uint64
- uintptr
- unsafe.Pointer

这些函数提供的原子操作共 5 种：
- 增或减
- 比较并交换
- 载入
- 存储
- 交换

不同原子操作适应场景有所区别

### 增或减



# 附加

悲观锁/乐观锁



自旋锁（乐观锁的一种）

什么是自旋锁？

自旋锁有啥问题？

 - ABA 问题？如何解决（加版本号）？
    - 在乎变化次数的，记录版本变更历史
    - 不在乎变化次数的，可以使用 true/false 标记
 - 更严重的问题：比较和更新是否能保证原子性（cas）
