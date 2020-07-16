## Bootstrap

环境准备：
- golang 环境
- gdb

测试环境版本：
```
➜  ~ go version
go version go1.13.12 linux/amd64
➜  ~
➜  ~ lsb_release -a
LSB Version:	n/a
Distributor ID:	ManjaroLinux
Description:	Manjaro Linux
Release:	20.0.3
Codename:	Lysia
➜  ~
➜  ~ gdb --version
GNU gdb (GDB) 9.2
```

GDB 使用

参考：[Debugging Go Code with GDB](https://golang.org/doc/gdb)

- 使用以下内容创建 test.go 文件
    ```go
    package main

    import (
        "fmt"
    )

    func main() {
        fmt.Println("hello, world")
    }
    ```
- 编译：`go build -gcflags "-N -l" -o test test.go`
- 使用 gdb 
    - 使用 gdb debug test：`gdb test`
    - 查看 test 的入口：`info files`
    - 在入口出设置断点：`b '{断点位置}'` （此处入口位置是："Entry point: 0x454e90" 中的 `0x454e90`）
    ```bash
    ➜  ~/work/github.com/beanscc/go-study/source/examples git:(master) ✗ gdb test
    Reading symbols from test...
    Loading Go Runtime support.
    (gdb) info files
    Symbols from "/home/beanscc/work/github.com/beanscc/go-study/source/examples/test".
    Local exec file:
        `/home/beanscc/work/github.com/beanscc/go-study/source/examples/test', file type elf64-x86-64.
        Entry point: 0x454e90
        0x0000000000401000 - 0x000000000048d083 is .text
        0x000000000048e000 - 0x00000000004dd5d5 is .rodata
        0x00000000004dd7a0 - 0x00000000004de40c is .typelink
        0x00000000004de410 - 0x00000000004de460 is .itablink
        0x00000000004de460 - 0x00000000004de460 is .gosymtab
        0x00000000004de460 - 0x00000000005496f8 is .gopclntab
        0x000000000054a000 - 0x000000000054a020 is .go.buildinfo
        0x000000000054a020 - 0x0000000000557118 is .noptrdata
        0x0000000000557120 - 0x000000000055e170 is .data
        0x000000000055e180 - 0x00000000005799f0 is .bss
        0x0000000000579a00 - 0x000000000057c168 is .noptrbss
        0x0000000000400f9c - 0x0000000000401000 is .note.go.buildid
    (gdb) b *0x454e90
    Breakpoint 1 at 0x454e90: file /opt/go/src/runtime/rt0_linux_amd64.s, line 8.
    (gdb)
    ```

查看文件 /opt/go/src/runtime/rt0_linux_amd64.s
```
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

#include "textflag.h"

TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
	JMP	_rt0_amd64(SB) // 这里跳转到了 TEXT _rt0_amd64(SB)

TEXT _rt0_amd64_linux_lib(SB),NOSPLIT,$0
	JMP	_rt0_amd64_lib(SB)
```

第 8 行，执行跳转到了 `_rt0_amd64(SB)`,  gdb 定位：
```
(gdb) b _rt0_amd64
Breakpoint 2 at 0x4514c0: file /opt/go/src/runtime/asm_amd64.s, line 15.
```

查看文件：`/opt/go/src/runtime/asm_amd64.s` 15 行：
```
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ	0(SP), DI	// argc
	LEAQ	8(SP), SI	// argv
	JMP	runtime·rt0_go(SB) // 跳转到了 runtime·rt0_go
```

这里又跳到了 `runtime·rt0_go(SB)`， gdb 再次定位：
> 注意:源码文件中的 "·" 符号编译后变成正常的 "."

```bash
(gdb) b runtime.rt0_go
Note: breakpoints 5, 6 and 7 also set at pc 0x4514d0.
Breakpoint 8 at 0x4514d0: file /opt/go/src/runtime/asm_amd64.s, line 89.
```

查看文件：`/opt/go/src/runtime/asm_amd64.s` 89 行：

```
TEXT runtime·rt0_go(SB),NOSPLIT,$0
...
DATA	runtime·mainPC+0(SB)/8,$runtime·main(SB)
GLOBL	runtime·mainPC(SB),RODATA,$8
```

至此，由汇编针对特定平台实现的引导过程就完成了，后面基本上都是由 golang 代码实现了