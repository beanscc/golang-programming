## 声明（declarations）



声明语句定义了程序的各种实体对象以及部分或全部的属性

go 主要由以下 4 种声明语句：

- `var` 声明变量
- `const` 声明常量
- `type` 声明类型或接口
- `func` 声明函数



一个 go 编写的程序由一个或多个以 `.go` 为文件后缀的源文件组成。

`.go` 源文件的内容依次应该是：

- 包的声明语句：`.go` 源文件应该以包的声明语句开始（声明语句前可以有注释），以说明该源文件是属于哪个包

- import 依赖导入声明语句：包声明语句之后是 import 语句，import 导入当前 `.go` 文件代码所依赖的其他包

- 包级的 变量/常量/函数/类型等声明语句：这部分的声明语句顺序无关紧要（注意：函数内部的名字，必须先声明后使用）

    

```
// Boiling prints the boiling point of water.
package main

import "fmt"

const boilingF = 212.0

func main() {
	var f = boilingF
	var c = (f - 32) * 5 / 9
	fmt.Printf("boiling point = %g°F or %g°C\n", f, c)
	// Output:
	// boiling point = 212°F or 100°C
}

```



以上示例种， `const boilingF` 属于包一级别的范围声明语句声明的，`f` 和 `c`是在 `main` 函数内部声明的，是局部声明。包一级声明语句声明的名字，可以在包内每个源文件中都可以访问；而局部声明的名称就只能在声明的那个小范围内被访问，关于名称的作用域，稍后见



有关 类型/接口 和 函数的声明在，后面介绍
