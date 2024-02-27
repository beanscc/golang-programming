本章节内容是跟着 “雨痕” 的 “golang 源码剖析" 第五版学习的内容


环境信息

操作系统：
```bash
root@618e10e1594b /h/y/w/golang# cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"
```


Go 版本：
```bash
root@618e10e1594b /h/y/w/golang# go version
go version go1.5.1 linux/amd64
```

Go env:

```bash
root@618e10e1594b /h/y/w/golang# go env
GOARCH="amd64"
GOBIN=""
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/home/yan/work/golang"
GORACE=""
GOROOT="/home/yan/work/golang/go"
GOTOOLDIR="/home/yan/work/golang/go/pkg/tool/linux_amd64"
GO15VENDOREXPERIMENT=""
CC="gcc"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0"
CXX="g++"
CGO_ENABLED="1"
```



Gdb

```bash
root@618e10e1594b /h/y/w/golang# gdb --version
GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-120.el7
```



找入口



1. 写个简单的 test.go

```go
package main
func main() {
    println("hello, world!")
}
```

2. 编译

    ```bash
    go build -gcflags "-N -l" -o test test.go
    ```

3. 使用 gdb 运行编译好的二进制文件

    ```bash
    gdb test
    ```

4. `info files` 查看二进制文件入口

    ```bash
    root@618e10e1594b /h/y/w/g/s/test# gdb test
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-120.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
    and "show warranty" for details.
    This GDB was configured as "x86_64-redhat-linux-gnu".
    For bug reporting instructions, please see:
    <http://www.gnu.org/software/gdb/bugs/>...
    Reading symbols from /home/yan/work/golang/src/test/test...done.
    warning: Missing auto-load scripts referenced in section .debug_gdb_scripts
    of file /home/yan/work/golang/src/test/test
    Use `info auto-load python [REGEXP]' to list them.
    (gdb)
    (gdb)
    (gdb) info files
    Symbols from "/home/yan/work/golang/src/test/test".
    Local exec file:
    	`/home/yan/work/golang/src/test/test', file type elf64-x86-64.
    	Entry point: 0x4561d0
    	0x0000000000401000 - 0x00000000004b2240 is .text
    	0x00000000004b3000 - 0x000000000053b4db is .rodata
    	0x000000000053b4e0 - 0x000000000053d588 is .typelink
    	0x000000000053d588 - 0x000000000053d588 is .gosymtab
    	0x000000000053d5a0 - 0x000000000058ece7 is .gopclntab
    	0x000000000058f000 - 0x0000000000590bc8 is .noptrdata
    	0x0000000000590be0 - 0x0000000000593150 is .data
    	0x0000000000593160 - 0x00000000005b6d38 is .bss
    	0x00000000005b6d40 - 0x00000000005bbb80 is .noptrbss
    	0x0000000000400fc8 - 0x0000000000401000 is .note.go.buildid
    (gdb)
    ```

    

5. `b *0x4561d0` 

    ```bash
    (gdb) b *0x4561d0
    Breakpoint 1 at 0x4561d0: file /usr/local/go/src/runtime/rt0_linux_amd64.s, line 8.
    ```

    

    