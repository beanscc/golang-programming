# golang-programming

Golang programming 学习

golang 官网(https://go.dev)

golang playground (https://go.dev/play/)

重点关注以下几部分文档：
- Effective Go (https://go.dev/doc/effective_go.html)
- Diagnostics (https://go.dev/doc/diagnostics.html)
- Frequently Asked Questions (FAQ) (https://go.dev/doc/faq)
- Package Documentation (https://go.dev/pkg)
- Command Documentation (https://go.dev/cmd/)
- Language Specification (https://go.dev/ref/spec)
- The Go Memory Model (https://go.dev/ref/mem)



**Contents**

- Get Started 
    - [安装](install.md)
- 基础
    - [packages](basics/basic/packages.md)
    - [命名](basics/basic/names.md)
    - [声明和作用域](basics/basic/declarations-and-scope.md)
    - 变量
    - 赋值
    - 类型
    - 包和文件

- 控制结构
    - 流程控制
        - if else
        - switch
    - 循环控制 for
    - goto
    - defer

- 数据结构

    - [bool](basics/types/boolean.md)

    - [number](basics/types/numeric.md)

    - [string](basics/types/string.md)
    
    - [operators](basics/types/operators.md)


- Aggregate types
    - [array](aggregate-types/array.md)
    - [struct](aggregate-types/struct.md)
- Reference types
    - [pointer](reference-types/pointer.md)
    - [slice](reference-types/slice.md)
    - [map](reference-types/map.md)
    - [function](reference-types/function.md)
    - [channel](reference-types/channel.md)
- [interface](interface/interface.md)
- [reflection](reflect/reflect.md)
- error
- defer and panic
    - [defer](errors/defer.md)
    - [panic](errors/panic.md)
- Concurrency
    - [context](concurrency/context.md)
    - sync
        - [Lock](sync/lock.md)
            - Mutex
            - RWMutex
            - Cond
        - [Map](sync/map.md)
        - [Pool](sync/pool.md)
        - [Once](sync/once.md)
        - [WaitGroup](sync/waitgroup.md)
    - [timer](time/timer.md)
    - [channel](channle/channel.md)
        - channel
        - select
    - [concurrency-patterns](concurrency/patterns.md)
- [scheduler](scheduler/scheduler.md)
    gpm 模型
- gc
    - [内存分配](memery-and-gc/memery-allocator.md)
    - [gc](memery-and-gc/memery-allocator.md)
    - [栈内容](memery-and-gc/stack-allocator.md)
- go程序是如何启动的
    - [bootstrap](init/bootstrap.md)
    - [init](init/init.md)
- profiling
    - [profiling](profiling/profiling.md)
    - [trace](profiling/trace.md)
- go command
    - go build -X
    - go tool
    - go test/benchmark 见’单元测试‘
    - go mod 见‘包管理工具 module’
    - go vet 相关见‘代码检查’
- [单元测试](command/test.md)
- [包管理工具 module](command/module.md)
- [代码检查](command/lint.md)
- 包
    - std 标准包
        - [flag](packages/std/flag.md)
        - [json](packages/std/json.md)
- go 查看用户代码对应汇编代码和runtime源码
- [附录](appendix)
    - [csv 文件操作和常见问题处理](appendix/csv.md)
    - wire 依赖注入：https://github.com/google/wire
    - [singleflight](appendix/singleflight.md)

- 遇到的问题
    - [fmt panic](problem/race-fmt-panic.md)
- 其他文章
    - [standard-go-project-layout](https://github.com/golang-standards/project-layout) [译](posts/standard-go-project-layout.md)

