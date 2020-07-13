# Go Concurrency Patterns: Timing out, moving on

- doc: https://blog.golang.org/go-concurrency-patterns-timing-out-and

并发编程都有自己的习语。一个很好的例子是超时。尽管 go 语言的 channel 不直接支持超时，但是很容易实现。

假设我们想从channel ch 中接受值，但是又想等待最多一秒之后才到达。我们首先创建一个信号 channel，并启动一个 goroutine 在给这个channel 发送数据之前，先 sleep 一秒

```go
timeout := make(chan boo, 1)
go func(){
    time.Sleep(1 * time.Second)
    timeount <- true
}()
```

我们可以使用 `select` 语句从 `ch` 或 `timeout` 中接受值。如果一秒之后 `ch` 通道中还接收不到数据，`select` 语句将选中 `timeout` 这个分支，并且放弃从 `ch` 尝试获取数据

```go
select {
case <-ch:
    // a read from ch has occurred
case <-timeout:
    // the read from ch has timed out
}
```

timeout 通道是一个只能容纳一个值通道，即非缓冲通道，它允许处理超时的那个 goroutine 向它发送值然后退出那个 goroutine，这个 goroutine 不知道也不关心发送的值是否被接收到。这意味着如果 ch 接收这个值在超时发生之前，这个 goroutine 不会一直阻塞下去。timeout 这个通道终将被垃圾回收机制回收并重新分配
（在上面例子中，我们使用了 time.Sleep 来演示 goroutine 和 channel 的机制。在真是编程场景下，应该使用 time.After， 一个函数，返回一个channel，并且在特定时间之后向这个channel发送数据）

让我们看看这种模式的另外一种变化。这个例子中，我们有一个程序可以同时从多个复制数据库中读取数据，该程序只需要其中一个答案，它应该接受首先到达的答案。



