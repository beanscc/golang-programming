## Strings, bytes, runes and characters in Go

参考：
- https://blog.golang.org/strings

### Code points, characters, and runes

至今为止，我们一直非常小心使用`字节`和`字符`。一部分原因是字符串包含字节，另一部分原因是处于字符有点难以表达的想法吧。Unicode 标准使用术语 代码点 来表示由单个值表示的项目。代码点 U+2318, 十六进制是 2318，表示符号⌘

选择一个更平淡的例子，Unicode 代码点U+0061 表示的是小写拉丁字母 'A':a


通常字符可以由许多不同的代码点序列表示，因此可以由 UTF-8 字节的不同序列标示


因此，计算中的字符概念含糊不清，或者至少是令人困惑的，所以我们使用时候需要谨慎。为了使事情是可靠的，有一些规范化的技术可以保证给定字符总是由相同的代码点来表示，但是这个主题离我们目前的主题有点远啦。后面的博客将解释 GO 库如何解决规范化问题。

`代码点` 稍微有点拗口，所以 go 给这个概念引入了一给简短的术语： `rune`。它出现在 go 库中和源代码中，与 代码点 完全相同，只是多了一个有趣的补充

go 语言定义 `rune` 为 `int32` 类型的别名，所以当一个整数值代表一个代码点时程序可以更清晰。而且，你可能会想到一个字符常量在 go 中被称为 `rune 常量`。 表达式 '⌘' 的类型是 `rune` 类型，值是 `0x2318`

总结一下就是一下几点：
- go 源码总是 UTF-8 编码格式
- 一个字符串包含了一些字节
- 一个字符串字面量，缺少字节级转义，但总是包含有效的 UTF-8 编码序列
- go 中不保证字符串中的字符被规范化

针对以上，我的理解是，一个字符串其实是由一个或多个字节组成的，一个字符串字面量，即便是不是由字节组成，也是包含有效的 UTF-8 编码的

### 范围循环

无须过多说明，go 源代码就是 UTF-8 编码的，go 对待 UTF-8 特别的地方只有一个，就是使用 `for range` 来循环一个字符串时候


上面我们已经见过常规的 `for` 循环对 rune 的操作。这里作为对比，我们使用 `for range` 循环来迭代一个 UTF-8 编码的 `rune` 值。每次循环时，循环的索引位置是当前 rune 的其实位置，以字节为单位，代码点是其值。

> Printf 函数的 %=U 参数，可以表示代码点的 Unicode 值及其表示的字符

示例：

```go
const nihongo = "日本語"
for index, runeValue := range nihongo {
    fmt.Printf("%#U starts at byte position %d\n", runeValue, index)
}

// 以下是输出结果：
// U+65E5 '日' starts at byte position 0
// U+672C '本' starts at byte position 3
// U+8A9E '語' starts at byte position 6
```

### 库

go 标准库中提供了用来解读 UTF-8 编码的强大支持，最重要的一个包是 `unicode/utf8`，它包含帮助程序来验证，反汇编和重新组装 UTF-8 字符串

以下示例代码，使用了 `utf8.DecodeRuneInString()` 这个函数的返回值是 rune 和 其宽度，以 UTF-8编码的字节为单位

```go
const nihongo = "日本語"
for i, w := 0, 0; i < len(nihongo); i += w {
    runeValue, width := utf8.DecodeRuneInString(nihongo[i:])
    fmt.Printf("%#U starts at byte position %d\n", runeValue, i)
    w = width
}

// 以下是输出结果：；
// U+65E5 '日' starts at byte position 0
// U+672C '本' starts at byte position 3
// U+8A9E '語' starts at byte position 6
```

### 总结

字符串是从字节切片构建的，所以通过索引访问会返回字节，而不是字符。字符串甚至可能不包含字符，事实上，字符的概念是模糊的，试图通过定义字符串由字符组成这个概念来解释是错误的