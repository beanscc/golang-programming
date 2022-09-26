## why fmt panic

遇到问题：TODO

同样问题的case: https://github.com/golang/go/issues/39587

先看下会导致这个问题的一个场景代码：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fullPath := "init"

	go func() {
		for {
			request(fullPath)
		}
	}()

	for {
		fullPath = ""
		time.Sleep(100 * time.Nanosecond)
		fullPath = "/test/test/test"
		time.Sleep(1000 * time.Nanosecond)
	}
}

func request(c string) {
	fmt.Printf("fullPath: %s", c)
}
```

运行结果：

```
fullPath: /test/test/test
fullPath: /test/test/test
fullPath: /test/test/test
....
fullPath: /test/test/test
fullPath: /test/test/test
fullPath: /test/test/test
fullPath: 
fullPath: /test/test/test
fullPath: /test/test/test
fullPath: /test/test/test
fullPath: /test/test/test
...
fullPath: /test/test/test
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x105cd26]

goroutine 18 [running]:
fmt.(*buffer).writeString(...)
        /usr/local/go/src/fmt/print.go:82
fmt.(*fmt).padString(0xc000092ad0, 0x0, 0xf)
        /usr/local/go/src/fmt/format.go:110 +0x8c
fmt.(*fmt).fmtS(0xc000092ad0, 0x0, 0xf)
        /usr/local/go/src/fmt/format.go:359 +0x61
fmt.(*pp).fmtString(0xc000092a90, 0x0, 0xf, 0xc000000073)
        /usr/local/go/src/fmt/print.go:450 +0x1ba
fmt.(*pp).printArg(0xc000092a90, 0x10af000, 0xc00008e5c0, 0x73)
        /usr/local/go/src/fmt/print.go:698 +0x843
fmt.(*pp).doPrintf(0xc000092a90, 0x10d1a33, 0xc, 0xc00009bfb8, 0x1, 0x1)
        /usr/local/go/src/fmt/print.go:1030 +0x15a
fmt.Fprintf(0x10ee3a0, 0xc0000ac008, 0x10d1a33, 0xc, 0xc0000407b8, 0x1, 0x1, 0xa, 0x0, 0x0)
        /usr/local/go/src/fmt/print.go:204 +0x72
fmt.Printf(...)
        /usr/local/go/src/fmt/print.go:213
main.request(...)
        /Users/yan/work/t-race/main.go:26
main.main.func1(0xc00008e1d0)
        /Users/yan/work/t-race/main.go:13 +0xa6
created by main.main
        /Users/yan/work/t-race/main.go:11 +0x6a

Process finished with exit code 2
```

先说下这个问题导致的原因：无锁竞争条件变量并发读写导致的
怎么检查呢？ race 检查，详见官方blog：https://blog.golang.org/race-detector

race 检查上面示例代码：

```
➜ go run -race main.go                                                                             
==================
WARNING: DATA RACE
Read at 0x00c000012200 by goroutine 7:
  main.main.func1()
      /Users/yan/work/t-race/main.go:13 +0x3c

Previous write at 0x00c000012200 by main goroutine:
  main.main()
      /Users/yan/work/t-race/main.go:18 +0xaf

Goroutine 7 (running) created at:
  main.main()
      /Users/yan/work/t-race/main.go:11 +0x92
==================
.... 程序运行输出省略 ....
```
