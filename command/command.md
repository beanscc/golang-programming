# commands

- doc:https://golang.google.cn/cmd/



## go build

compile packages and dependencies

usage:
```
go build [-o output] [-i] [build flags] [packages]
```

— ldflags 链接参数

```bash
#!/bin/bash
buildTime=`date +%FT%T%z`
commit=`git rev-parse --short HEAD`

# -ldflags 参数说明 https://golang.google.cn/cmd/link/
go build -ldflags "-X main.gitHash=$commit -X main.buildTime=$buildTime" main.go
```

- gcflags 编译参数

```
# 打印出汇编信息
go tool compile -S -N -l main.go

# 指定编译参数 -S -N -l 
go build -gcflags "-S -N -l" main.go
```

## go get

add dependencies to current module and install them


## go mod

module maintenance
for more about modules, see `go help modules`


# 包依赖管理工具vgo

go get 查看文档：go help get
go module 查看文档：go help modules