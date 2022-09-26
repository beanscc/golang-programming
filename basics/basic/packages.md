## packages

每个 go 程序都是由若干 `package` 组成

程序入口在一个 `main` 的 package 中



```go
package main

import (
	"fmt"
	"math/rand"
)

func main() {
	fmt.Println("My favorite number is", rand.Intn(10))
}

```



按照约定，package 的名称和导入路径的最有一个元素一样。如，`"math/rand"` 包包含以语句 `rand` 开头的文件



go 语言针对的处理单元是`包`而不是文件。这样我们可以将包拆分成任意数量的文件。

在 go 编译器看来，如果所有这些文件的包声明都是一样的，那么他们就同样属于一个包，这根把所有内容放在一个单一的文件里是一样的。

以上，所以，通常我们可以根据应用程序的功能将其拆分成尽可能多的包，以保持一切`模块化`

按照惯例，`package` 的描述信息应该放在 `package` 声明之前

包引用需要注意的一些用法

- 使用 `_` 占位包别名，只引用相应得包，一般这样做，是为了调用包中得初始化函数

```go
import "database/sql"
import _ "github.com/go-sql-driver/mysql"

db, err := sql.Open("mysql", "user:password@/dbname")
```

- 使用英文小数点，省略包别名，就可以引入当前目录名相应的包，且在使用时，不用在前面加上包名（不建议使用，容易混淆）

```go
import (
	. "bufio" // 注意前面英文小数点
)

func readBytes(buf *Reader) string {
    // ...
}
```





### 可见性/可导性

在 go 中，若**名称以大写字母开头**，则该名称 `对外可见`，也称为 `可导出`。其他的均`不可导出`

如 `Pi` 就是可导出的, 相反 `pi` 就不可导出

当 导入一个包时，仅能访问可导出的名称，任何不可导出的名称都无法从包外部进行访问



## imports



在一个 package 中使用另一个 package 的代码，首先需要使用 import 关键字导入目标 package，格式如上面示例中的:

```go
import (
	"fmt"
    "math/rand"
)
```



也可以拆分开来，每行一个 import , 如

```go
import "fmt"
import "math/rand"
```

但，一般使用 `()` 分组使用一个 import 即可



### import 使用别名

包名和已有 import 中包名冲突时，可使用别名

```go
import (
	"git/path/util"
    myutil "my/path/util"
)
```



特殊别名 `_` ，表示只是引入这个包，在当前代码中不需要调用这个包的任何代码，一般情况这么做只是需要这个包触发一下 `init()` 函数, 如

```go
package db

import (
	_ "github.com/go-sql-driver/mysql"
)
```



这里通过 `_ "github.com/go-sql-driver/mysql"` 方式引入这个包，仅仅是需要触发这个包的 `init()` 函数，完成 mysql 的驱动注册

```go
// 文件：github.com/go-sql-driver/mysql/driver.go
func init() {
	sql.Register("mysql", &MySQLDriver{})
}
```