# golang-programming
Golang programming 总结回顾夯实基础

关于 go 的更多官方信息，参见[golang.org](https://golang.org) (国内可以参见[golang.google.cn](https://golang.google.cn))

- [go doc](https://golang.google.cn/doc/)

重点关注以下几部分文档：
- Effective Go
- Diagnostics
- Frequently Asked Questions (FAQ)
- Package Documentation
- Command Documentation (https://golang.google.cn/cmd/)
- Language Specification (https://golang.google.cn/ref/spec)
- The Go Memory Model (https://golang.google.cn/ref/mem)

# Contents

- [安装](install.md)
- [程序结构](程序结构.md)     
- Basic types
    - [布尔类型](basic-types/bool.md)
    - [数值类型](basic-types/number.md)
    - [字符串类型](basic-types/string.md)
- Aggregate types
    - [数组](aggregate-types/array.md)
    - [struct](aggregate-types/struct.md)
- Reference types
    - [pointer](reference-types/pointer.md)
    - [slice](reference-types/slice.md)
    - [map](reference-types/map.md)
    - [function](reference-types/function.md)
    - [channel](reference-types/channel.md)
- [Interface](interface/interface.md)
- [Reflection](reflect/reflect.md)
- defer and panic
    - [defer](defer-panic/defer.md)
    - [panic](defer-panic/panic.md)
- Concurrency
    - [context](context/context.md)
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
- [scheduler](scheduler/scheduler.md)
    gpm 模型
- gc
    - 内存分配
    - gc
    - 栈内容
- 标准包
    - [json](packages/json.md)