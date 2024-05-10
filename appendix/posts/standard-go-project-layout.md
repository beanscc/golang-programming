# Standard Go Project Layout

- 相关文章：https://github.com/golang-standards/project-layout/blob/master/README.md

## Go 工程根目录

### /cmd

工程的主应用目录

每个应用的目录命令，应该保持和你想要的应用名称一致（如：`/cmd/myapp`）

不要在应用目录放入很多代码。如果你任务这些代码可以被外部引用或被其他应用所使用，那么这些代码应该放在`/pkg`目录。如果这部分代码是不可重复用的，或者不想被其他人重用，那么将其放到 `/internal` 目录。你不会知道其他人会如何使用的你代码，所以请明确你的意图！

通常有一个很小的 main 函数，它可以导入和调用 `/internal` 和 `/pkg` 目录下的代码而不是其他内容

示例：

```bash
# prometheus 项目
prometheus/cmd/prometheus/main.go
```

### /internal

私有应用和依赖库代码。这个目录下的代码是你不想被其他人在他们的应用或库中引用的

将你实际应用的代码放入 `/internal/app` 目录（如：`/internal/app/myapp`），并且这部分代码可以被 `/internal/pkg` 目录下的应用所使用(如： `/internal/pkg/pyprivlib`)

### /pkg

库代码是可以被外部应用所使用的（如：`/pkg/mypubliclib`）。其他项目可以引用这些库代码，并其他它们可以使用，因此在你将代码放入这个目录前，请三思而行

当你的工程根目录包含很多非 go 的组件和目录时，在一个地方将代码分组，也是一种方式，它可以很容易的运行各种 go tool

如果你想了解哪些流行的 go repo 使用了这种布局方式，请查看 [/pkg](https://github.com/golang-standards/project-layout/blob/master/pkg/README.md) 目录。这是一种常用的布局方式，但并非一种被普遍接收的方式，并且 go 社区的一些人并不推荐这种方式。

### /vendor

应用依赖目录（手动管理或者使用你喜欢的依赖管理工具，如：[dep](https://github.com/golang/dep)）

如果你在构建一个库，请不要提交应用程序依赖代码

## 服务应用目录

### /api

OpenAPI/Swagger 规范，json 数据结构定义文件, protocol buffer 定义文件

示例（[更多示例](https://github.com/golang-standards/project-layout/blob/master/api/README.md)）：

```bash
├── kubernetes
│   ├── api-rules
│   │   ├──  README.md
│   │   └──  violation_exceptions.list
│   ├── openapi-spec
│   │   ├──  README.md
│   │   ├──  BUILD
│   │   └──  swagger.json
...
```

## Web 应用目录

### /web

web 应用特定组件：静态 `assets`；服务端模版 `templates`；单页 web 应用(single page web application, `SPA`)

## 常用应用目录

### /configs

配置文件模版或默认配置文件。

可以将 `confd` 或 `consul-template` 模版文件放在这里

### /init

系统初始化(systemd, upstat, sysv)和进程管理器（runit, supervisord）配置

### /scripts

脚本目录，用于执行各种构建，安装，分析等操作

这些脚本可以让项目根目录的 Makefile 尽量的小而简单（如：https://github.com/hashicorp/terraform/blob/master/Makefile）

示例（[更多示例](https://github.com/golang-standards/project-layout/blob/master/scripts/README.md)):

```bash
├── helm
│   ├── scriptes
│   │   ├──  coverage.sh
│   │   └──  sync-repo.sh
|   └── Makefile
```

### /build

打包和持续集成

将你的云（AMI），容器（Docker），OS（deb/rpm/pkg）打包配置和脚本放在 `/build/package` 目录

将你的 CI(travis/circle/drone) 配置和脚本放在 `/build/ci` 目录。注意有些 CI 工具对配置文件的位置有要求。如果可以的话，请尝试将配置文件放在 `/build/ci` 目录，通过软链接的方式，将他们链接到 CI 工具要求的位置

### /deployments

laaS, PaaS, 系统和容器编排部署配置和模版（docker-compose, kubernetes/helm, mesos, terraform, bosh）

### /test

额外的外部测试应用和测试数据。你可以随意构建 `/test` 目录。对于更大的项目，有必要设置一个数据子目录。例如，如果需要 Go 忽略该目录中的内容，可以使用 `/test/data` 或 `/test/testdata` 目录。请注意， Go 也会忽略以 `.` 或 `_` 开头的目录或文件，因此在命令测试数据目录时，更具灵活性

[更多示例](https://github.com/golang-standards/project-layout/blob/master/test/README.md)

## 其他目录

### /docs

设计和用户文档（除了你的 godoc 生成的文档）

[更多示例](https://github.com/golang-standards/project-layout/blob/master/docs/README.md)

### /tools

当前项目的支持工具。请注意，这些工具可以引用来自 `/pkg` 和 `/internal` 包中的码

[更多示例](https://github.com/golang-standards/project-layout/blob/master/tools/README.md)

### /examples

你的应用或公开库的使用示例

[更多示例](https://github.com/golang-standards/project-layout/blob/master/examples/README.md)

### /third_party

外部辅助工具，forked 代码和其他第三方实用程序（如：Swagger UI）

### /githooks

git hooks

### /assets

和你的库一起使用的其他静态材料（如：图片，logo 等等）

### /website

这里可以放项目的网站数据

[更多示例](https://github.com/golang-standards/project-layout/blob/master/website/README.md)

## 不应该有的目录

### /src

有些 Go 项目有 `src` 目录，但这通常发生于来自 Java 开发者的项目中，在 Java 中这是一种常见模式。如果你可以帮助自己尝试不采用此 Java 模式。你不会希望你的 Go 代码或者 Go 项目看起来像 Java

不要将项目级别 `/src` 目录和 Go 工作空间目录 `/src` 目录混淆。`$GOPATH` 环境变量指向你当前的 Go 工作空间（在非 windows 系统上，默认情况下，它指向 `$HOME/go` 目录）。Go 工作空间包括顶级 `/pkg`， `/bin`， `/src` 这几个目录。你的实际项目最终应该成为 `/src` 目录下的一个子目录，因此如果项目中有 `/src` 目录，项目路径看起来将是这样的 `$GOPATH/src/your_project/src/your_code_dir`。请注意，Go 1.11 版本可以将项目放在 `$GOPATH` 之外，但这并不意味着使用此布局是个好主意
