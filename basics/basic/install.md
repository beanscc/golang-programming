## 安装

安装参见：https://go.dev/doc/install

linux源码安装说明：
- 下载 Go 安装源码包
- 解压到需要安装的目录
- 设置环境变量
- 查看 go 版本并编码验证安装的正确性

下载go源码安装包[download go](https://go.dev/dl/)

解压到指定位置（eg：/usr/local），将会在指定位置目录下创建 go 目录树（eg：/usr/local/go）

```bash
tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz
```

添加环境变量PATH，向 `/etc/profile` 或 `$Home/.profile` 或 `$Home/.bashrc` 或 `$Home/.zshrc` 添加以下export语句
```bash
export PATH=$PATH:/usr/local/go/bin
```

> 多个环境变量之间分隔在各平台不同（linux 平台下用`:`分隔； windows平台中以 `;` 分隔）

go 默认的工作区目录是 `$Home/go`, 若想使用不同的目录，需要设置 `GOPATH` 环境变量(即若未明确设置 `GOPATH` 环境变量，则 `GOPATH` 使用的是默认目录 `$Home/go`)

测试安装是否正确

```bash
# 查看 go 安装版本
go version 
# 我本地当前输出：go version go1.13.7 darwin/amd64

# 查看 go 的相关环境变量，含GOROOT/GOPATH/GOPROXY 等
go env
```

以上正确输出的话，说明go安装基本成功了，当然还需要编码、编译、运行来确认安装无误   
进入 `$GOPATH` 目录，创建 `src/hello` 目录，然后进入 hello 目录，创建 `hello.go` 文件，输入以下代码
```go
package main

import "fmt"

func main() {
	fmt.Printf("hello, world\n")
}
```

使用 go build 进行编译
```bash
$ cd $GOPATH/src/hello
$ go build
```

以上命令将在当前目录编译产生一个名为hello的可执行的二进制文件，执行该文件，将看到打印文本
```bash
$ ./hello
hello, word
```