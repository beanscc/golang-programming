# GMP 调度模型

- 参考文档：https://golang.org/s/go11sched
- https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit



名次说明

G
M
P



M0

G0





主 goroutine 入口：

https://github.com/golang/go/blob/go1.14.15/src/runtime/proc.go#L113







## G



G 的状态：

- `_Gidle`: 表示这个 goroutine 刚刚被分配，还未初始化
- `_Grunnable`: 表示这个 goroutine 在一个 run queue 中， 它当前没有执行用户代码，没有栈的所有权
- `_Grunning`: 表示这个 goroutine 可以运行用户代码，当前 goroutine 有栈的所有权。它不在 run queue 中。他被分配一个 M 和一个 P（g.m 和 g.m.p 是有效的）
- `_Gsyscall`: 表示这个 goroutine 正在执行一个系统调用。它没有执行用户代码。它有栈的所有权。它不在 run queue 中，它被分配一个 M
- `_Gwaiting`: 表示 goroutine 在运行时被阻塞。它没有执行用户代码。它不在 run queue 中，但应该记录在某处（如，channel 等待队列），以便在需要时准备就绪。除了 channel 操作可以在适当的channel 锁下读或者写栈的部分之外，它没有栈的所有权。否则，在 goroutine 进入此状态后，访问栈是不安全的（如，它可能会被移动）
- `_Gdead`: 表示这个 goroutine 当前未使用。它可能刚刚退出，在 free list 中，或刚刚被初始化。它没有执行用户代码。它可能有也可能没有分配栈。G 和它的栈（若有的话）由退出 G 或者从  free list 获得 G 的 M 所有
- `_Gcopystack`: 表示这个 goroutine 的栈正在被移动。它没有执行用户代码。它不在 run queue 中。该栈由将其放入 _Gcopystack 状态的 goroutine 所有
- `_Gpreempted`: 表示这个 goroutine因为一个 suspendG 抢占暂停了自己。他就像 `_Gwaiting`，但还没有任何东西负责它准备就绪。某些 suspendG 必须原子的比较和交换状态到 `_Gwaiting` 状态，以便它准备就绪时进行响应
- `_Gscan`: 结合上述除  `_Grunning`  之外的状态之一，表示 GC 正在扫描栈。它没有执行用户代码，栈由将其设置为 `_Gscan` 的 goroutine 所有
- `_Gscanrunning`:  _Gscanrunning 不同，它用于在 GC 通知 G 扫描其自己的栈时短暂地阻止状态的转换。否则就像 _Grunning



## P

P 的状态：

- `_Pidle`: 表示没有使用 P 来运行用户代码或调度程序。通常它在空闲 P list 中，并可供调度程序使用，但他可能只是在其他状态之间转换。P 由空闲 P list 或正在转换其状态的任何内容拥有。它的 run queue 是空的
- `_Prunning`: 表示 P 由 M 拥有且用于运行用户代码或调度程序。只有拥有这个 P 的 M 才被允许从`_Prunning` 状态改变 P 的状态。 M 可以将 P 转换为 `_Pidle`（若没有更多工作要做）、`_Psyscall`（当进入系统调用时）、`_Pgcstop`（因为 GC 而暂停时）。M 也可以将 P 的所有权直接移交给另一个 M （如，调度一个有锁的 G）
- `_Psyscall`: 表示 P 没有运行用户代码。它与系统调用中的 M 有亲缘关系，但不属于它，并且可能被另一个 M 偷走。这与 `_Pidle` 类似，但使用轻量级转换并维护 M 的亲缘关系。要离开 `_Psyscall` 状态必须通过 CAS 完成，无论是为了窃取还是夺回 P。请注意，存在 ABA 风险：即使 M 在系统调用成功后将其原始 P 通过 CAS 返回到 `_Prunning`，也必须了解 P 可能在此期间已被另一个 M 使用过
- `_Pgcstop`: 表示 P 因为 STW 暂停，并由 STW 的M所拥有。STW 的 M 继续使用它的 P，即使在 `_Pgcstop` 状态。从 `_Prunning` 到 `_Pgcstop` 的转换会导致 M 释放其 P 并停放。 P 保留它的 run queue，而 startTheWorld 将在具有非空 run queue 的 P 上重新启动调度程序
- `_Pdead`: 表示 P 不在使用（GOMAXPROCS 缩小）。若 GOMAXPROCS 增大，我们用户重用 Ps。一个死的 P 大部分都被剥夺了它的资源，尽管还有些一些东西（如，跟踪缓冲区）

