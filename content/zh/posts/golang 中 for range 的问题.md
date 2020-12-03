---
title: golang 中 for range 的问题
tags:
  - golang
url: 75.html
id: 75
categories:
  - golang
  - 编程杂记
date: 2019-04-17 14:21:09
---

最近项目在使用 golang 开发，对于一直使用 c 开发的我来说 golang 有着十分强大的内置库，生态也还可以，使用起来确实十分的舒适，最重要的是不需要手动管理内存，让人感觉快乐了很多，哈哈。不过，在写项目的时候也遇到了些因为不熟悉语言特性而出现的问题，比如 for range。代码逻辑大致如下:

```golang
type foo struct {
    a string
    b string
}

func main() {
    var F = []foo{ 
       foo{"1", "2"},
       foo{"3", "4"},
    }

    M := make(map[string]*foo)
    for _, v := range F {
        M[v.a] = &v
    }

    log.Printf("%v\n", M)
}
```



这段代码的本意 是想遍历切片 F 并把其元素的地址赋给 map M。但是运行的时候发现最后 M 中所有的 value 都是相同的值。重新 review 一边之后，发现在 13~15 行，我一直都在把临时变量 v 的地址赋给 M，而 for 循环都结束之后，v 的值是 F 中最后一个元素，所以 M 中所有的 value 就都一样了，且都是 F 中的最后一个元素。

为了避免这个问题，可以在 for 循环中在创建一个临时变量来保存 F 中的元素，并将其赋值给 M：

```golang
for _, v := range F {
    tmp = v
    M[v.a] = &tmp
}
```

或者不要使用 for range 的形式：

```golang
for i := 0; i < len(F); i++ {
    M[F[i].a] = &F[i]
}
```