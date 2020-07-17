# golang-programming
Golang programming 总结回顾夯实基础

关于 go 的更多信息，参见官网[golang.org](https://golang.org) (国内可以参见[golang.google.cn](https://golang.google.cn))

重点关注以下几部分文档：
- Effective Go (https://golang.org/doc/effective_go.html)
- Diagnostics (https://golang.org/doc/diagnostics.html)
- Frequently Asked Questions (FAQ) (https://golang.org/doc/faq)
- Package Documentation (https://golang.org/pkg)
- Command Documentation (https://golang.org/cmd/)
- Language Specification (https://golang.org/ref/spec)
- The Go Memory Model (https://golang.org/ref/mem)

# Contents

- [安装](install.md)
- [程序结构](程序结构.md)     
- Basic types
    - [bool](basic-types/bool.md)
    - [number](basic-types/number.md)
    - [string](basic-types/string.md)
- Aggregate types
    - [array](aggregate-types/array.md)
    - [struct](aggregate-types/struct.md)
- Reference types
    - [pointer](reference-types/pointer.md)
    - [slice](reference-types/slice.md)
    - [map](reference-types/map.md)
    - [function](reference-types/function.md)
    - [channel](reference-types/channel.md)
- [Interface](interface/interface.md)
- [Reflection](reflect/reflect.md)
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
    - [内存分配](gc/memery-allocator.md)
    - [gc](gc/memery-allocator.md)
    - [栈内容](gc/stack-allocator.md)
- go程序是如何启动的
    - [bootstrap](init/bootstrap.md)
    - [init](init/init.md)
- [profiling](profiling/profiling.md)
- [单元测试](command/test.md)
- [包管理工具 module](command/module.md)
- [代码检查](command/lint.md)
- 标准包
    - [flag](packages/flag.md)
    - [json](packages/json.md)
- 其他文章
    - [standard-go-project-layout](https://github.com/golang-standards/project-layout) [译](posts/standard-go-project-layout.md)