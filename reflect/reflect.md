# Reflect



[反射三大定律](laws-of-reflection.md)

1. 从 interface 变量到反射对象的转换

2. 从反射对象到 interface 变量的转换

3. 要修改一个反射对象，则这个值必须是可设置的



定律1:

reflect 包中有 2 个重要的对象，Type 和 Value 

以下两个函数提供了 interface 变量到反射对象的转换

- reflect.TypeOf() refelect.Type 

- reflect.ValueOf() reflect.Value		

    

Type 主要用于变量类型方面的操作

Value 主要用于访问或设置变量的值，同时也提供了获取 Type 的方法 （value.Type()）



定律2:

通过 Value 结构的 Interface() 方法可以重新获取 interface{} 对象

可以通过以下方式判断：

- Value.CanInterface() ：方法表示 Value 值是否可以调用 Interface() 方法而不产生 panic



定律3:

go 中函数参数都是值传递的，即传入的都是参数的副本，这时候若在函数作用域内改变这个副本，并不会影响到函数/方法外的本体。

指针传递，若传入的是参数地址的引用（其实就是参数的地址值），那么传入的就是参数地址值的副本，这样在函数作用域内修改此地址所指向的值，则任何可以访问此地址的地方所访问到的值都会是变化后的值



以上同样适用在 反射 中。所以若需要在反射中修改变量结构字段的值，首先需要判断当前这个变量是否可设置，可以通过以下方式判断：

- Value.CanSet() : 表示 Value 值是否可以被修改，只有 Value 是可寻址的，且不是通过不可导出结构体字段获得时，才是可以被修改的，否则调用 Set 或者任何特定类型的 setter 方法都将会 panic









