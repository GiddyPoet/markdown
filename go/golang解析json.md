#### Golang处理JSON---编码

##### 数据结构map对应关系

数据类型 | JSON |Golang
:-:|:-:|:-:
字串 | string | string
整数 | number | int64
浮点数 | number | float64
数组 | array | slice
对象 | object | struct
布尔 | bool | bool
空值 | null |nil

##### 基本结构编码
1. 首先定义json结构体
2. 使用Marshal方法序列化

定义结构体根据golang语言隐藏特性，只有字段名首字母大写才会编码到json中
```golang
package main

import (
	"encoding/json"
	"fmt"
	"log"
)

type Account struct {
	Email    string
	password string
	Money    float64
}

func main() {
	account := Account{
		Email:    "GiddyPoet@gmail.com",
		password: "123456",
		Money:    100.5,
	}

	rs, err := json.Marshal(account)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(rs)
	fmt.Println(string(rs))
}
```

注：
* byte 等同于int8，常用来处理ascii字符
* rune 等同于int32,常用来处理unicode或utf-8字符
* 如果需要取rune的长度，可以采用len([]rune(str)) 或 utf8.RuneCountInString(str)


##### 复合结构编码
该编码结构无非是增加了slice切片，或者是map等结构，和之前的方式一样。

##### 嵌套编码
slice和map可以匹配json的数组和对象，当前提是对象的value是同类型的情况。更通用的做法，对象的key可以是string，但是其值可以是多种结构。

```golang
package main

import (
	"encoding/json"
	"fmt"
	"log"
)

type User struct {
	Name    string
	Age     int
	Roles   []string
	Skill   map[string]float64
	Account Account
}

type Account struct {
	Email    string
	password string
	Money    float64
}

func main() {
	skill := make(map[string]float64)

	skill["python"] = 99.5
	skill["elixir"] = 90
	skill["ruby"] = 80.0
	account := Account{
		Email:    "GiddyPoet@gmail.com",
		password: "123456",
		Money:    100.5,
	}
	user := User{
		Name:    "rsj217",
		Age:     27,
		Roles:   []string{"Owner", "Master"},
		Skill:   skill,
		Account: account,
	}

	res, err := json.Marshal(user)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(string(res))
}
```
**通过定义嵌套的结构体Account，实现了key与value不一样的结构。golang的数组或切片，其类型也是一样的，如果遇到不同数据类型的数组，则需要借助空结构来实现**
```golang
type User struct {
    ...

    Extra []interface{}
}

extra := []interface{}{123, "hello world"}

user := User{
    ...
    
    Extra:   extra,
}
```
注：通过定义接口类，来处理不同数据类型的数组的编解码

```golang
type User struct {
    Name    string
    Age     int
    Roles   []string
    Skill   map[string]float64
    Account Account

    Extra []interface{}

    Level map[string]interface{}
}

通过接口类，来处理多种不同的值，go中的nil对应着json中的null
func main() {

    ...

    level := make(map[string]interface{})

    level["web"] = "Good"
    level["server"] = 90
    level["tool"] = nil

    user := User{
        Name:    "rsj217",
        Age:     27,
        Roles:   []string{"Owner", "Master"},
        Skill:   skill,
        Account: account,
        Level:   level,
    }

    ...
}
```


##### StructTag 字段重名
通过上面的例子，我们看到了Level字段中的keyserver等是小写字母，其他的都是大写字母。因为我们在定义结构的时候，只有使用大写字母开头的字段才会被导出。而通常json世界中，更盛行小写字母的方式。看起来就成了一个矛盾。其实不然，golang提供了struct tag的方式可以重命名结构字段的输出形式。

```golang
type Account struct {
    Email    string  `json:"email"`
    Password string  `json:"pass_word"`
    Money    float64 `json:"money"`
}

output:
{"email":"rsj217@gmail.com","pass_word":"123456","money":100.5}
```
通过上述方式可以在json编码后将json重命名。


##### 忽略字段
通常使用marshal的时候，会把结构体的所有除了私有字段都编码到json，而实际开发中，我们定义的结构可能更通用，我们需要某个字段可以导出，但是又不能编码到json中。

```golang
type Account struct {
    Email    string  `json:"email"`
    Password string  `json:"-"`
    Money    float64 `json:"money"`
}
```

注：特别的是指,当前Password字段可以选择在json编码的时候不导出，可以用"-"的方式忽略。

##### 可选字段omitempty
```golang
type Account struct {
    Email    string  `json:"email"`
    Password string  `json:"password,omitempty"`
    Money    float64 `json:"money"`
}
func main() {
    account := Account{
        Email:    "rsj217@gmail.com",
        Password: "",
        Money:    100.5,
    }
    ...
}
```
注：omitempty是指，如果该字段赋值了则就输出，如果没有复制则就不编码至json输出中。


##### string选项
```golang
type Account struct {
    Email    string  `json:"email"`
    Password string  `json:"password,omitempty"`
    Money    float64 `json:"money,string"`
}
func main() {
    account := Account{
        Email:    "rsj217@gmail.com",
        Password: "123",
        Money:    100.50,
    }

    ...
}
```

注：通过上述的string选项，可以将float64在json显示中由数字类型转换成string类型。

##### 总结

上面所介绍的大致覆盖了golang的json编码处理。总体原则分两步：
1. 首先定义需要编码的结构。
2. 然后调用encoding/json标准库的Marshal方法生成json byte数组，转换成string类型即可。

golang和json的大部分数据结构匹配，对于复合结构，go可以借助结构体和空接口实现json的数组和对象结构。通过struct tag可以灵活的修改json编码的字段名和输出控制。

----

#### Golang处理JSON----解码

解码步骤大体分为:
1.定义结构
2.反序列化
```golang
type Account struct {
    Email    string  `json:"email"`
    Password string  `json:"password"`
    Money    float64 `json:"money"`
}

var jsonString string = `{
    "email":"rsj217@gmail.com",
    "password":"123",
    "money":100.5
}`

func main() {

    account := Account{}

    err := json.Unmarshal([]byte(jsonString), &account)
    if err != nil{
        log.Fatalln(err)
    }
    fmt.Printf("%+v\n", account)
}
```


匹配查找的原则：
与编码类似，golang会将json的数据结构和go的数据结构进行匹配。匹配的原则就是寻找tag的相同的字段，然后查找字段。查询的时候是大小写不敏感的


```golang
type Account struct {
    Email    string  `json:"email"`
    PassWord string  
    Money    float64 `json:"money"`
}
```
上述结构，可以输出{Email:rsj217@gmail.com PassWord:123 Money:100.5}，把Password改为PassWord同样可以支持，但是如果**结构的字段是私有的，即使tag符合，也不会被解析**

```golang
type Account struct {
    Email    string  `json:"email"`
    password string  `json:"password"`
    Money    float64 `json:"money"`
}
```
上述方式无论如何都不会解析password的字段，因为其是不可导出的。

##### string tag
在编码的时候，我们使用tag string，可以把结构定义的数字类型以字串形式编码。同样在解码的时候，只有字串类型的数字，才能被正确解析，或者会报错。

##### tag
与编码一样，tag的-也不会被解析，但是会初始化其零值。


#### 动态解析
1.通过NewDecoder创建一个从r读取并解码json对象的*Decoder，解码器有自己的缓冲，并可能超前读取部分json数据。
原型：func NewDecoder(r io.Reader) *Decoder
2.在调用NewDecoder方法Decode，Decode从输入流读取下一个json编码值并保存在v指向的值里。
原型：func (dec *Decoder) Decode(v interface{}) error
```golang
type User struct {
    UserName string `json:"username"`
    Password string     `json:"password"`
}

var jsonString string = `{
    "username":"rsj217@gmail.com",
    "password":"123"
}`

func Decode(r io.Reader)(u *User, err error)  {
    u = new(User)
    err = json.NewDecoder(r).Decode(u)
    if err != nil{
        return
    }
    return
}

func main() {
    user, err := Decode(strings.NewReader(jsonString))
    if err !=nil{
        log.Fatalln(err)
    }
    fmt.Printf("%#v\n",user)
}
```
**注：因为NewReader的入参是一个stream流，可以通过strings.NewReader()方法将string变成流。**
func NewReader(s string) *Reader
NewReader创建一个从s读取数据的Reader

##### 接口
通过golang的断言，将空接口的类型进行转换
```golang
type User struct {
    UserName interface{} `json:"username"`
    Password string      `json:"password"`
}

func Decode(r io.Reader) (u *User, err error) {
    u = new(User)
    if err = json.NewDecoder(r).Decode(u); err != nil{
        return
    }
    switch t := u.UserName.(type) {
    case string:
        u.UserName = t
    case float64:
        u.UserName = int64(t)
    }
    return
}

func main() {
    user, err := Decode(strings.NewReader(jsonString))
    if err != nil {
        log.Fatalln(err)
    }
    fmt.Printf("%#v\n", user)
}
```
上述通过go的断言，将类型进行了转换。

上述go语言的断言的方式：
```golang
value, ok := a.(string)
if !ok {
    fmt.Println("It's not ok for type string")
    return
}
```
或者采用switch语句进行断言
```golang
var t interface{}
t = functionOfSomeType()
switch t := t.(type) {
default:
    fmt.Printf("unexpected type %T", t)       // %T prints whatever type t has
case bool:
    fmt.Printf("boolean %t\n", t)             // t has type bool
case int:
    fmt.Printf("integer %d\n", t)             // t has type int
case *bool:
    fmt.Printf("pointer to boolean %t\n", *t) // t has type *bool
case *int:
    fmt.Printf("pointer to integer %d\n", *t) // t has type *int
}
```