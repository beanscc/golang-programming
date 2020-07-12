## encoding/json

本章节介绍 json 相关只是


### Unmarshal 遇见 null

有一点我们需要了解：标准包解析 `null` 时，并不会报错，即 err == nil
当需要解析的字符串是 `null` 时, 需要注意，json.Unmarshal 对目标对象的指针二次引用，会导致该对象的类型值被初始化为指针的零值（即 `nil`）


```go
package main

import (
	"encoding/json"
	"fmt"
)

type Person struct {
	Name string `json:"name"`
}

func main() {
	// 正常 json 数据解析
	b1 := []byte(`{"name": "Mr D"}`)
	p1 := Person{}
	err := json.Unmarshal(b1, &p1)
	fmt.Printf("err=%v, p1=%#v \n", err, p1)
	// output: err=<nil>, p1=&main.Person{Name:"Mr D"}

	// 结构错误的 json 解析
	b2 := []byte(`{"name": 1}`)
	p2 := Person{}
	err2 := json.Unmarshal(b2, &p2)
	fmt.Printf("err=%v, p2=%#v \n", err2, p2)
	// output: err=json: cannot unmarshal number into Go struct field Person.name of type string, p2=&main.Person{Name:""}

	// 结构错误的 json 解析, 且指针二次引用
	b21 := []byte(`{"name": 1}`)
	p21 := &Person{Name: "p21"}
	err21 := json.Unmarshal(b21, &p21)
	fmt.Printf("err=%v, p21=%#v, p21: %p \n", err21, p21, p21)
	// output: err=json: cannot unmarshal number into Go struct field Person.name of type string, p21=&main.Person{Name:"p21"}, p21: 0xc0000102f0

	// 正常结构，但字段出现 null
	b3 := []byte(`{"name": null}`)
	p3 := &Person{Name: "p3"}
	fmt.Printf("before json.Unmarshal. p3=%#v, p3: %p \n", p3, p3) // before json.Unmarshal. p3=&main.Person{Name:"p3"}, p3: 0xc000010330
	err3 := json.Unmarshal(b3, &p3)
	fmt.Printf("err=%v, p3=%#v, p3: %p \n", err3, p3, p3)
	// output: err=<nil>, p3=&main.Person{Name:"p3"}, p3: 0xc000010330

	// only null 且解析时指针二次引用
	b4 := []byte(`null`)
	p4 := &Person{Name: "p4"}                                      // 注意这里是指针类型
	fmt.Printf("before json.Unmarshal. p4=%#v, p4: %p \n", p4, p4) // before json.Unmarshal. p4=&main.Person{Name:"p4"}, p4: 0xc000010390
	err4 := json.Unmarshal(b4, &p4)                                // 注意：这里是对指针的二次引用
	fmt.Printf("err=%v, p4=%#v, p4: %p \n", err4, p4, p4)
	// output: err=<nil>, p4=(*main.Person)(nil), p4: 0x0
	// 由于对 p4 这个指针的二次指针引用，导致 p4 的类型值从 Person类型的零值，变成了 nil
    // 所以若后面再类似调用 p4.Name 将导致程序 panic

	// only null
	b5 := []byte(`null`)
	p5 := Person{Name: "p5"}
	fmt.Printf("before json.Unmarshal. p5=%#v, p5: %p \n", p5, &p5) // before json.Unmarshal. p5=main.Person{Name:"p5"}, p5: 0xc0000103c0
	err5 := json.Unmarshal(b5, &p5)
	fmt.Printf("err=%v, p5=%#v, p5: %p \n", err5, p5, &p5)
	// output: err=<nil>, p5=main.Person{Name:"p5"}, p5: 0xc0000103c0

	// 指针/引用类型 仅仅进行声明，未进行初始化时，其地址均指向一个空地址:0x0
	var iv *int
	var fn func()
	var mp map[string]string
	fmt.Printf("iv:%p, fn:%p, mp:%p \n", iv, fn, mp)
	// output: iv:0x0, fn:0x0, mp:0x0
}
```

### Unmarshal 时指针的二次引用

```go
package main

import (
	"encoding/json"
	"log"
)

type Person struct {
	Name string `json:"name"`
}

func main() {
	var iv *int
	pp := &Person{}
    b := []byte(`null`)
	err := json.Unmarshal(b, &pp) // err == nil，且由于对pp这个指针的二次指针引用，导致 pp 的类型值从 Person类型的零值，变成了 nil
	log.Printf("err=%v, pp=%#v, pp: %p, iv:%p", err, pp, pp, iv) // 2020/03/04 23:26:12 err=<nil>, pp=(*main.Person)(nil), pp: 0x0, iv:0x0
}
```