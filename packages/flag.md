# flag 标准库-命令行参数解析

flag 参数不支持 `ls -ll` 这样在`-` 后面有多个标志参数。每个参数必须是分开的。
不区分 `-` 和 `--`，所以 `-flag` 和 `--flag` 是等价的

#### flags 的定义方式：
- （1） flag.Xxx() ， 返回相应类型的指针， eg：
    ```go
    var ip = flag.Int("flagname", 123, "help message for flagname")
    ```
- （2） flag.XxxVar(), 将flag绑定到一个变量上，eg:
    ```go
    var flagvar int
    flag.IntVar(&flagvar, "flagname", 1232, "help message for flagname")
    ```

> 另外，也可以创建自定义flag，只要实现flag.Value接口即可

> 在所有flag 定义完成之后，可以通过调用flag.Parse()进行解析

#### 命令行flag的语法
- （1） -flag       // 只支持bool类型    
- （2） -flag=x     
- （3） -flag x    // 只支持非bool 类型   

> Parse() 中，对bool类型进行了特殊处理，默认的，提供了-flag，则对应的值为true，否则flag.Bool/BoolVar中指定的默认值；如果希望显示设置为false则使用-flag=false  
    ```
    # Eg:
    ./hooker_in_om -port ":8112" -mode "release" -token "http://ticket.devel.wesai.com/index.php?r=user/getcuruser" -api "http://ticket.devel.wesai.com" -encry=false -ida "http://ticket.devel.wesai.com/index.php?r=item/detail/id/"
    ```
