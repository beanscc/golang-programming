# install



参考：https://go.dev/doc/install



## download

从 go 官方站点（ https://go.dev/dl/）下载适用于你系统的版本进行安装



## install



从 Go 1.13 开始，`go` 命令默认会通过 Google 官方提供的 Go 模块镜像（proxy.golang.org）和校验和数据库（sum.golang.org）来下载和校验依赖包。这些服务会参与你的依赖管理过程。

- 你可以在 [https://proxy.golang.org/privacy](vscode-file://vscode-app/private/var/folders/2w/xh1g95vn7mn0c2k5wz18lktw0000gn/T/AppTranslocation/23BE5262-E011-4CC9-B16A-DDED550F6271/d/Visual Studio Code.app/Contents/Resources/app/out/vs/code/electron-browser/workbench/workbench.html) 查看这些服务的隐私政策。
- 如果你不想用 Google 的这些服务，或者想用其他代理，可以通过配置（如设置 `GOPROXY` 环境变量）来关闭或更换这些服务，具体方法见 Go 官方文档。

### linux



1. 下载合适的 go 

   ```bash
   wget https://go.dev/dl/go1.25.2.linux-arm64.tar.gz
   ```

2. 删除之前安装的 go，将 go1.25.2 解压至 /usr/local 目录，将会在 /usr/local/go 目录下全新的目录树

   ```bash
    rm -rf /usr/local/go && tar -C /usr/local -xzf go1.25.2.linux-amd64.tar.gz
   ```

3. 添加 PATH 环境变量

   ```bash
   export PATH=$PATH:/usr/local/go/bin
   ```

   将其添加到 `$HOME/.profile` 或 `/etc/profile`，要使环境变量立即生效，可执行

   ```bash
   source $HOME/.profile
   ```

4. 执行以下命令，验证 Go 是否安装成功，检查输出是否刚安装的 Go 版本

   ```bash
   go version
   ```

   

### mac

1. 下载 go 安装程序

   ```bash
   wget https://go.dev/dl/go1.25.2.darwin-amd64.pkg
   ```

2. 打开上面下载的 go 安装程序，并根据引导提示完成安装

   安装引导程序默认会将 Go 安装到 `/usr/local/go` 目录，安装程序通常会自动把 `/usr/local/go/bin` 目录加到你的 `PATH` 环境变量中

3. 执行以下命令，验证 Go 是否安装成功，检查输出是否刚安装的 Go 版本

   ```bash
   go version
   ```

   



## 附录

### GOPATH 是否需要设置

在 Go Modules 模式下( go1.11 开始引入， go 1.13 开始成为官方推荐的依赖管理方式，并默认开启），一般不需要手动配置 `GOPATH` 环境变量，Go 会自动管理依赖和缓存

- 如果你用的是 **Go Modules**（项目目录下有 go.mod），`GOPATH` 只影响本地缓存目录（默认在 `~/go`），不影响依赖管理和开发流程。
- 只有在**老项目**（未启用 Go Modules）或需要自定义缓存目录时，才需要配置 `GOPATH`。
