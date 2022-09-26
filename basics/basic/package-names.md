## Package names

包命名

- doc: https://blog.golang.org/package-names

go 代码是组织在 `package` （包）内的。在一个包内部，代码可以引用任何内部定义的变量；而对于包的引用客户端来讲，它就只能引用包定义的可导出类型、函数、常量及变量。诸如此类的引用总是以包名称为前缀的。例如： foo.Bar 这个引用来讲，就是引用的名称为 foo 的包里面一个命名为 Bar 的变量

一个好的包命名会使代码更好。包名为包的内容提供上下文，可以让引用它的客户端更容易理解包的用途和如何使用它。包名称也可以帮助包的维护者决定什么可以放入包中，什么不放入包中。良好的包命名可以让你更容易地找到所需的代码


Effective Go 提供了命名包、类型、函数、变量的指南。本文对该讨论进行了扩展，并对标准库中的名称进行了调查。还讨论了错误的包名称以及如何解决

### 包命名

好的包名称是简短、明了的。

包命名规则如下：
- 简单的名词
- 全部小写
- 不使用下划线形式（no under_scores）
- 不使用驼峰形式 （no mixedCaps）

经常是简单的名词，如下

- time （提供用于计算和展示时间的函数）
- list （实现双向链表）
- http （提供了HTTP客户端和服务端的实现）

其他语言中的命名方式可能在go语言中不是提倡的方式。以下两种命令方式可能在其他语言中是良好的命名形式，但却不符合go语言的良好形式：

- computeServiceClient
- priority_queue


一个 go 包可提供多种可导出类型和函数。举个栗子：一个名为 `compute` 的包，可能提供一个 `Client` 类型及其方法，用于使用服务的方法以及用于跨多个客户端划分计算任务的函数

**明智的缩写**

当程序员熟悉缩写时，包名可以缩写。广泛使用的包通常是有缩写名称的，如下：
- strconv (string conversion)
- syscall (system call)
- fmt (formatted I/O)

但，另一方面，若缩写包名使其含糊不清，那就不要使用缩写

`不要从用户哪里窃取好的名称`。要避免包的名称经常被客户端用来当作变量名。举个栗子：提供缓冲I/O功能的包 `bufio`，为啥不叫 `buf`，因为 `buf` 对于 buffer 来说是一个很好的变量名称

### 包内容命名

包名称及其内容的名称是联系在一起的，因为客户端代码是一起使用它们的。 在设计包时，请从客户端的角度出发考虑一下

**避免口吃**

啥意思呢，就是要避免包名和包变量名称中的重复。因为客户端代码要使用包的内容时，需要使用包名称作为前缀的，所以在包内命名时不要重复包名。HTTP Server 由 `http` 包提供，叫作 `Server`，而不是 `HTTPServer`。客户端代码引用时是这样的 `http.Server`，这也没有啥歧义

**简单明了的函数名称**

放包 pkg 中函数返回 pkg.Pkg 类型的值时，函数名称通常可以省略类型名称而不会产生混淆，如下：


```go
start := time.Now()   // start 是一个time.Time 类型
t, err := time.Parse(time.Kitchen, "6:06PM") t 是 time.Time 类型

ctx = context.WithTimeout(ctx, 10*time.Millisecond)  // ctx 是 context.Context
ip, ok := userip.FromContext(ctx)  //  ip 是 net.IP
```

一个命名为 New() 的函数，在 pkg 包中，返回的是 pkg.Pkg 类型，这是使用该数据类型的客户端代码的标准入口点

```go
p := pkg.New()  // p 是一个新的 pkg.Pkg 类型
```

当一个函数返回的值类型是 pgk.T 而 T 又不是 Pkg，这个函数名可以包含 T ，使客户端代码更容易理解。
比较常见的场景是一个包中含有多个 New 这种形式的函数，如下：
```go
d, err := time.PraseDuration("10s") // d 是一个 time.Duration类型
elapsed := time.Since(start)  // elapsed 是 time.Duration 类型
ticker := time.NewTicker(d) // ticker 是 *time.Ticker 类型
timer := time.NewTimer(d) // timer 是 *time.Timer 类型
```

相同的类型可以在存在于不同的包中，因为从客户端引用来看，所有类型前面都有包名为前缀。
举个栗子🌰：标准包中包含好几种名为 Reader 的类型，包括 jpeg.Reader, bufio.Reader, csv.Reader。每个包名都适合 Reader，以产生一个好都类型名称

如果你不能给一个包的内容取一个有意义的包名，那么这个包的抽象化就是失败的。使用你编写的包来编写客户端代码，若看起来很糟糕，请重构你的抽象出来的包中代码。这种方式将使你的包更容易被客户端理解和包的开发者所维护

### 包路径

go 包是有名称和路径的。
包名是在包源码文件中声明的。
客户端代码使用包名作为包内容引用的前缀，当 import 时才使用包路径。按惯例，包路径的最后一个元素就是包名

```go
import (
    "golang.org/x/time/rate"  // package rate
)
```

go 构建工具将包路径映射到文件 GOPATH 目录下的 src 目录中。 go 工具使用环境变量 GOPATH 目录来查找 go 源码文件（举个栗子：go build 工具将在 `$GOPATH/src/github.com/user/hello` 文件目录下查找 `github.com/user/hello` 这个包）

**目录**

标准包使用类似目录 `crypto`, `container`, `encoding` 和 `image` 来组织关于算法和协议的go包。这些目录中go 包之间没有实际的关系。目录只提供了一种排列文件的方法。任何包都可以导入其他包，只要不创建循环导入（啥意思呢，就是说 package A 引用了 package B，package B 又引用了 package A，包之间相互引用，就会出现循环）


就像不同数据类型在不同包中可以使用相同的名称一样，在不同的目录结构下，可以使用相同的包名，是没有歧义的。
举个栗子， `runtime/pprof` 以 `pprof` 分析工具所需的格式提供分析数据，而 `net/http/pprof` 提供 HTTP 端点以此格式显示分析数据（runtime/pprof提供数据， net/http/pprof 展示数据）。客户端代码使用包路径导入包，因此没有困惑。若某个源代码文件需要同时引用 `pprof` 包，可以在引用时可以对其中一个或两个包使用包别名。当使用别名时，命名别名也需要遵循包命名规范（小写、不需要下划线方式、不要使用驼峰形式）

### 错误的包命名

错误的命名使代码难以导航和维护

以下是识别和修复错误的包命名的一些指导规范：
- 避免无意义的包名称
- 分解通用包
- 不要将所有API抽象到一个包中
- 避免不必要的包名冲突

下面来一一讲解上面的指导规范

**避免无意义的包名称**

包名为 `util`，`common`，`misc` 对于客户端来讲关于包中的内容是什么没有任何意义。这将使客户端更难以使用包，同样使包的维护人员难以保持包的关注。随着时间的推移，它们会累积依赖关系，这些依赖将使编译显著且不必要的慢，尤其在大型程序中。并且由于这些包名是通用的，它们更可能于客户端代码导入的其他包冲突，迫使客户端发明名称以区分它们


**分解通用包**

为了修复以上这种问题，可以将一些有一样名称的类型和函数提取出来组织关于它们的包。

举个栗子，比如你有以下包

```go
package util

func NewStringSet(...string) map[string]bool {...}
func SortStringSet(map[string]bool) []string {...}
```

客户端使用以上代码可能会像这样：
```go
set := util.NewStringSet("c", "a", "b")
fmt.Println(util.SortStringSet(set))
```

针对以上代码，我们可以将这部分代码从 util 包中提取出来，组织成一个新的包，选择一个能够表达其内容的包名，如下：
```go
package stringset

func New(...string) map[string]bool {...}
func Sort(map[string]bool) []string {...}
```

这样的话，客户端使用起来，就像下面这样了
```go
set := stringset.New("c","a","b")
fmt.Println(stringset.Sort(set))
```

一旦，你做出以上改变，那么将很容易看到如何优化新包的代码，如下：
```go
package stringset
type Set map[string]bool

func New(...string) Set {...}
func (s Set) Sort() []string {...}
```

这样的话，将产生更简洁的客户端代码，如下：
```go
set := stringset.New("c", "a", "b")
fmt.Println(set.Sort())
```

`包的名称是其设计的关键部分。要致力于消除项目中无意义的包名称`

**不要将所有API抽象到一个包中**

许多 **用心良苦** 的程序员，在他们的程序中，将所有公开的接口放到一个名为 `api` , `types`, `interfaces` 的包中，认为这样更容易发现他们代码的入口。`这其实是错误的`。这样的包和哪些被命名为 `util` 或者 `common` 的包有一样的痛点，没有边界的增长，对使用者来讲没有任何指导作用，累计依赖关系，和其他包产生碰撞。`请分解它们，可以使用目录去分离公共包的实现`

**避免不必要的包名冲突**

虽然在不同的目录中可以使用相同的包名，但是包名经常是一起使用的，所以需要具有不同的名称。这将减少困惑，同样减少客户端本地起别名的需求。基于同样的原因，避免使用和流行的标准包同名的包名，例如：io 或 http

### 总结

包名称是go语言中良好命名的关键。请花费时间去抉择良好的包名，组织良好的代码结构。这将有助于客户端理解和使用你的包，同时也有助于维护者优雅的维护它们