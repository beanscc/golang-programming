## Init

初始化过程主要完成以下：
- 命令行参数整理
- 环境变量设置
- 内存分配器
- 垃圾回收器
- 并发调度器
...

```go
// The main goroutine.
func main() {
    // 获取当前 Goroutine g, 编译器将为此函数的调用者重写指令，直接获取 g（从 TLS 或者从专用寄存器获取）
	g := getg()

	// Racectx of m0->g0 is used only as the parent of the main goroutine.
	// It must not be used for anything else.
	g.m.g0.racectx = 0

    // 执行栈最大限制是：64位架构上 1 GB ； 32位架构上 250MB
    // 使用十进制而不是二进制的 GB 和 MB 是因为十进制在栈溢出的失败消息中更好看一些
	if sys.PtrSize == 8 {
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}

	// Allow newproc to start new Ms.
	mainStarted = true

	if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
		systemstack(func() {  // 启动系统后台监控，定期垃圾回收，以及并发任务调度相关
			newm(sysmon, nil)
		})
	}
 
    // 在初始化期间将主 goroutine 锁定到主 OS 线程上
    // 大多数程序都不会在意，但有少部分确实需要主线程进行某些调用
    // 那些在初始化期间可以通过调用 runtime.LockOSThread 安排 main.main 在主线程中运行来保留锁
	lockOSThread()

	if g.m != &m0 {
		throw("runtime.main not on m0")
	}

    // runtime init 初始化
	doInit(&runtime_inittask) // must be before defer
	if nanotime() == 0 {
		throw("nanotime returning zero")
	}

	// Defer unlock so that runtime.Goexit during init does the unlock too.
	needUnlock := true
	defer func() {
		if needUnlock {
			unlockOSThread()
		}
	}()

	// Record when the world started.
	runtimeInitTime = nanotime()

	gcenable()

	main_init_done = make(chan bool)
	if iscgo {
		if _cgo_thread_start == nil {
			throw("_cgo_thread_start missing")
		}
		if GOOS != "windows" {
			if _cgo_setenv == nil {
				throw("_cgo_setenv missing")
			}
			if _cgo_unsetenv == nil {
				throw("_cgo_unsetenv missing")
			}
		}
		if _cgo_notify_runtime_init_done == nil {
			throw("_cgo_notify_runtime_init_done missing")
		}
		// Start the template thread in case we enter Go from
		// a C-created thread and need to create a new thread.
		startTemplateThread()
		cgocall(_cgo_notify_runtime_init_done, nil)
	}

    // 用户逻辑中的 init 函数初始化调用
	doInit(&main_inittask)

	close(main_init_done)

	needUnlock = false
	unlockOSThread()

	if isarchive || islibrary {
		// A program compiled with -buildmode=c-archive or c-shared
		// has a main, but it is not executed.
		return
    }
    // 用户逻辑 main 入口
	fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
	fn()
	if raceenabled {
		racefini()
	}

	// Make racy client program work: if panicking on
	// another goroutine at the same time as main returns,
	// let the other goroutine finish printing the panic trace.
	// Once it does, it will exit. See issues 3934 and 20018.
	if atomic.Load(&runningPanicDefers) != 0 {
		// Running deferred functions should not take long.
		for c := 0; c < 1000; c++ {
			if atomic.Load(&runningPanicDefers) == 0 {
				break
			}
			Gosched()
		}
	}
	if atomic.Load(&panicking) != 0 {
		gopark(nil, nil, waitReasonPanicWait, traceEvGoStop, 1)
	}

	exit(0)
	for {
		var x *int32
		*x = 0
	}
}
```

用户代码逻辑中，所有 init 函数执行完后，才会执行 main 函数

init 函数的执行顺序是：
- 先执行 main 包依赖包的所有 init 
- 最后执行 main 包的 init
- 同一文件中的多个 init 函数执行顺序与 init 函数体定义的位置先后顺序有关