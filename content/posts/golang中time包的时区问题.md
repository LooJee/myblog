---
title: golang 中 time 包的时区问题
tags:
  - golang
url: time_package_in_golang.html
id: time_package_in_golang
categories:
  - golang
date: 2019-05-24 16:16:49
---

最近项目中有一个功能，是定时从远端服务器同步数据到本地，数据中有一个字段是时间格式的。每次我的同步程序从服务器上获取到数据和本地数据库保存的数据进行比较的时候，总是会提示数据发生了变化，但是事实上我并没有修改服务器上的数据。

为了查出问题出现的原因，我在同步的时候将从服务器获取的数据和本地数据库读取的数据都打印了出来进行对比，发现两个数据唯一的区别在于时间的时区不同：服务器获取的数据是 UTC，而本地数据库的数据用的是 CST。而服务器上获取的时间本是字符串的，是我使用 time.Parse 函数将其转换成时间的，那么问题可能就出在这个函数身上了。

找到方向后，我就看了下 time.Parse 的源码，发现它实际是调用了 time.parse 函数，而 time.parse 函数的第三个参数就是需要转换的时间默认的时区，而 time.Parse 将该参数赋为 UTC，这可能就是产生错误的原因了。我尝试将 time.Parse 替换成了 time.ParseInLocation，问题解决。

![](http://www.tech-seeker.cn/wp-content/uploads/2019/05/image.png)



示例：

```golang
package main
import (
    "fmt"
    "time"
)

func main() {
     t := "2019-10-10 10:10:10"
     t1, _ := time.Parse("2006-01-02 15:04:05", t)
     t2, _ := time.ParseInLocation("2006-01-02 15:04:05", t, time.Local)
     fmt.Println(t1)
     fmt.Println(t2)
     fmt.Println(t1.Equal(t2))
}
```

输出结果：

![](http://www.tech-seeker.cn/wp-content/uploads/2019/05/image-1.png)

