---
title: 初识golang中的反射
date: 2019-09-13 14:26:53
tags:
  - golang
url: first_meet_golang_reflect.html
---

定义类型、声明变量、使用变量是一门编程语言的基本功能，我们可以这样来定义一个结构体类型：

```go
type Foo struct {
    X string `foo:"x"`
    Y int    `foo:"y"`
}
```

像这样来使用这个类型声明一个变量：

```go
var bar Foo
```

使用变量也很方便，像这样就可以在终端打印出结构体的字段了:

```go
fmt.Printf("%s, %s", bar.X, bar.Y)
```

以上这些操作都是在明确了变量的类型的时候进行的，不过在编程过程中，我们可能会遇到一种情况：在编写代码时无法明确变量的类型，变量的信息只有在程序运行时在会获取，比如这样的函数：

```go
func theFunc(val interface{})
```

它的函数签名只有一个参数类型为 interface 的参数 val，这意味着可以将任意类型的变量作为实参传入函数。这样的情况下该如何操作变量 val 呢？golang 标准库中有一个 reflect包--即反射--它提供了一系列的方法能帮助我们在运行时获取变量的信息，或者修改变量的值。在反射中有三个比较重要的概念：Type、Kind 和 Value。下面就一起来看看反射的奇妙之处吧。

如果我们把 bar 变量传入了 theFunc 函数，在函数中我们需要知道些什么信息呢？可能会需要知道传入的变量是什么类型的，如果是一个结构体，可能还会需要知道结构体中有那些字段，这些字段又是什么类型的。我们可能还需要根据结构体字段特定的 tag 来执行特定的操作，比如 encoding/json 包就会根据 tag 来给序列化的 json 字段取名。如果传入的变量是我们所期望的，我们可能还需要修改它的值，或者用它来创建一个新的变量。我们就一个个来谈谈如何用反射来实现这些功能把。

## 获取变量类型

### TypeOf()

reflect 提供了 TypeOf 函数来获取指定变量的类型，它的返回值类型为 reflect.Type，这是一个接口类型，它提供了一系列的方法来获取变量相关信息的方法。

### Name()、Kind() 和 Elem()

Name() 方法获取变量的类型名称，不过它有一个限制，即只能获取基本类型或者自定义的结构体的类型名称，其他的类型会返回一个空的字符串。

Kind() 方法会返回变量的内置类型名称，比如 ptr、slice、array、map、func、struct 等等。它通常可以和 switch 配合来做类型判断。

Elem() 方法用于判断类型的元素类型（type's element type）。它是对 Name 方法的补充，它可以返回 array, chan, map, ptr, 或 slice 类型中元素的类型。比如，针对一个指针 &bar， Elem 方法会返回 Foo 这个类型名称；针对 []string 这样一个字符串 slice，Elem 会返回 string 这个类型名称。如果不在允许的类型上调用 Elem 方法，会导致 panic ，其实 reflect 包中很多方法和函数都是这样的，它要求使用者知道自己在做什么。

下面来举一个完整的示例吧：

```go
import (
	"fmt"
	"reflect"
)

type Foo struct {
	X string  `foo:"x"`
	Y string `foo:"y"`
}

func main() {
	bar := Foo{
		X: "hello",
		Y: "world",
	}
	sli := make([]string, 0)
	ch := make(chan bool)
	m := make(map[int]int)
	arr := [10]int{}
	i := 0
	f := 1.1
	b := true

	theFunc(bar)
	theFunc(&bar)
	theFunc(sli)
	theFunc(ch)
	theFunc(m)
	theFunc(arr)
	theFunc(theFunc)
	theFunc(i)
	theFunc(f)
	theFunc(b)
}

func theFunc(val interface{}) {
	valType := reflect.TypeOf(val)
	fmt.Printf("name of value : %s, kind of value : %s, ", valType.Name(), valType.Kind())

	switch valType.Kind() {
	case reflect.Ptr, reflect.Array, reflect.Chan, reflect.Slice, reflect.Map:
		fmt.Printf("elem of value : %s", valType.Elem())
	}
	fmt.Println()
}
```

输出为：

![](https://raw.githubusercontent.com/LooJee/medias/master/images/20190913173656.png)

## 获取变量值

### ValueOf()

ValueOf() 函数获取变量中实际存储的值。例如：

```go
bar := Foo {
    X: "hello",
    Y: "world",
}

fmt.Println(reflect.ValueOf(bar))
```

输出的结果为:

```
{"hello", "world"}
```

### 类型断言

如果使用者知道传入的值是什么类型的，或者通过类型判断的方式获取了值的类型信息，那么可以使用 val.(type) 这种类型断言的方式将传入的 interface 类型的数据强制转换成我们所需要的类型，这时候就可以正常使用类型的字段了。需要注意的是，如果类型断言出错了，那么将会引发 panic，所以，在使用的时候可以先确认变量的类型再进行类型断言，或者利用类型断言的第二个布尔类型的返回值来判断断言是否成功。

```go
func main() {
	bar := Foo {
	    X: "hello",
	    Y: "world",
	}
    
    theFunc(bar)
}

func theFunc(val interface{}) {
    // v1 := val.(int) // 引发 panic
    //在使用断言前先判断类型是否正确
    if reflect.TypeOf(val) == reflect.TypeOf(Foo{}) {
        v2 := val.(Foo)
        fmt.Println(v2)
    }
    //使用 ok 来判断断言是否成功
    if v2, ok := val.(Foo); ok {
        fmt.Println(v2)
    }
}
```

输出为

```go
{"hello" "world"}
{"hello" "world"}
```

### 遍历类型的方法

NumMethod()、Method(int)、 MethodByName(string) 这几个方法可以获取变量所对应的类型**已导出**的方法：

```go
type Foo struct {
	X string  `foo:"x"`
	Y string `foo:"y"`
}

func (Foo) unExported() {
	fmt.Println("unExported")
}

func (Foo) Exported()  {
	fmt.Println("Exported")
}

func main() {
	bar := Foo{
		X: "hello",
		Y: "world",
	}

	valTheFun(bar)
}

func valTheFun(val interface{}) {
	valType := reflect.TypeOf(val)
	fmt.Println(valType.NumMethod())
	for i := 0; i < valType.NumMethod(); i++ {
		fmt.Println(valType.Method(i).Name)
	}
}
```

输出结果：

```go
1
Exported
```

### 遍历函数的入参和出参

对于函数类型，Type 接口也提供了相应的方法来遍历入参和出参：NumIn()，In(i int), NumOut，Out(i int)。这几个方法只能用与类型为函数的变量，否则将会引发 panic 。

```go
func main() {
    valTheFun(valTheFun)
	valTheFun(param)
}

func param(i int, s string) (int, string) {
	return i, s
}

func valTheFun(val interface{}) {
	valType := reflect.TypeOf(val)
	
    fmt.Printf("number of in args %d:\n", valType.NumIn())
	for i := 0; i < valType.NumIn(); i++ {
		fmt.Printf("\t %s\n", valType.In(i))
	}
	fmt.Printf("number of out args %d:\n", valType.NumOut())
	for i := 0; i < valType.NumOut(); i++ {
		fmt.Printf("\t %s\n", valType.Out(i))
	}
}
```

输出：

```go
number of in args 1:
	 interface {}
number of out args 0:
number of in args 2:
	 int
	 string
number of out args 2:
	 int
	 string
```

### 遍历结构体的字段

遍历结构体的字段是十分常用的，当我们需要统一处理 kind 为结构体的入参，但又不知道结构体的具体类型，也不需要知道结构体的具体类型的时候，就可以用上遍历结构体字段的一系列方法了。结构体中还有一个 tag 属性，用它可以给结构体中的字段添加额外的属性，golang 也提供了方法用于遍历 tag 的数据。个人觉得这一部分的功能还是需要好好研究一下的，熟悉这部分的操作可以写出更好的代码，golang 标准库中比较常用的场景有： encoding/json 将结构体序列化为 json 字符串；encoding/xml 将结构体序列化为 xml 数据等等。下面给一个简单的例子：

```go
func main() {
	bar := Foo{
		X: "hello",
		Y: "world",
	}

	structFunc(bar)
}

func structFunc(val interface{}) {
	valType := reflect.TypeOf(val)
	fmt.Println("number of fields in val :", valType.NumField())
	for i := 0; i < valType.NumField(); i++ {
		fmt.Printf("field : %s ", valType.Field(i).Name)
		fmt.Println("tags", valType.Field(i).Tag)
	}
}
```

输出：

```
number of fields in val : 2
field : X tags foo:"x"
field : Y tags foo:"y"
```

### 创建新的变量

reflect 中有两种函数可以创建新的变量，分别是 New(Type) 和 Make* 。用两种是因为 Make* 是一系列的函数，它们和内建函数 make 一样，只能为 slice, map,  chan 来创建新的变量，不同的是 reflect 为这几个类型都分别声明了一个 Make 函数，并且还为 func 类型也声明了一个 Make 函数。而 New 和内建的 new 函数一样，返回的是创建的变量的指针，这个很重要。因为给新建的变量设置值的时候，需要使用 Field() 方法来指定结构体中的字段，而这个方法的接收器必须为 struct，所以，新建的变量必须要先调用 Elem() 方法来获取对应的结构体类型，然后再调用 Field() 方法来设置新的值。

```go
func main() {
	bar := Foo{
		X: "hello",
		Y: "world",
	}

	valType := reflect.TypeOf(bar)
	valFields := reflect.ValueOf(bar)

	val := reflect.New(valType)
	//因为 val 是一个指针，所以需要使用 Elem 来获取元素的实际类型
	val.Elem().Field(0).SetString(valFields.Field(0).String())
	val.Elem().Field(1).SetString("golang")

	//val 是一个 reflect.Value 类型的变量，
    //需要通过 Interface() 来获取它所维护的数据，
    //然后再通过类型断言强制转换为指定的类型
	if v, ok := val.Interface().(*Foo); !ok {
		panic("wrong type")
	} else {
		fmt.Println(*v)
	}
}
```

输出：

```
{hello golang}
```

## 小结

到这里，基本是把 reflect 的基本操作都讲了一遍了，本来是想简单写写的，结果越写越多。可能有些方面没有将的十分清楚，请各位看官多多斧正。