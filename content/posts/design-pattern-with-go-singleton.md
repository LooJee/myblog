---
title: "Design Pattern With Go: Singleton"
tags:
  - golang
  - design-pattern
categories:
  - golang
  - design-pattern
date: 2021-03-22T11:58:09+08:00
---
## 定义

一个类只允许创建一个对象（或者实例），那这个类就是一个单例类，这种设计模式就叫作单例模式。当某些数据只需要在系统中保留一份的时候，可以选择使用单例模式。

## 饿汉式

饿汉式的实现方式比较简单。在类加载的时候，静态实例就已经创建并初始化好了，所以，实例的创建过程是线程安全的。如果实例占用资源多，按照 fail-fast 的设计原则（有问题及早暴露），那我们也希望在程序启动时就将这个实例初始化好。如果资源不够，就会在程序启动的时候触发报错，我们可以立即去修复。这样也能避免在程序运行一段时间后，突然因为初始化这个实例占用资源过多，导致系统崩溃，影响系统的可用性。

下面是一个典型的饿汉式单例模式：

```go
package singleton

import "sync/atomic"

type HungrySingleton struct {
	data uint64
}

var hungry = &HungrySingleton{}

func HungrySingletonInstance() *HungrySingleton {
	return hungry
}
```

## 懒汉式

懒汉式单例模式可以延迟加载类实例，如果想要在使用类实例的时候再创建，可以采用这种方式来实现单例模式。不过如果类实例初始化时间比较长，可能会对效率有一定的影响。

```go
package singleton

import (
	"sync"
	"sync/atomic"
)

type LazySingleton struct {
	data uint64
}

var lazySingleton *LazySingleton
var lock sync.Mutex

func init() {
	lock = sync.Mutex{}
}

func LazySingletonInstance() *LazySingleton {
	lock.Lock()
	defer lock.Unlock()
	if lazySingleton == nil {
		lazySingleton = &LazySingleton{}
	}
	return lazySingleton
}
```

## 双重检测懒汉式

上面面这种实现方式比较简单粗暴，每次获取实例的时候都需要加锁，这会大大增加时间开销，效率十分低下。我们可以改进下，创建实例的时候，先判断实例是否已经创建，如果没有创建再进入创建流程，这样会减少等锁的次数，增加效率。

```go
package singleton

import "sync"

type LockCheckLazySingleton struct {
	data int64
}

var ll sync.Mutex
var lcSingleton *LockCheckLazySingleton

func init() {
	ll = sync.Mutex{}
}

func LockCheckLazySingletonInstance() *LockCheckLazySingleton {
	if lcSingleton == nil {
		ll.Lock()
		defer ll.Unlock()
		if lcSingleton == nil {
			lcSingleton = &LockCheckLazySingleton{}
		}
	}
	return lcSingleton
}
```

在 golang 中，我们可以使用 sync.Once 来简化判断流程：

```go
package singleton

import (
	"sync"
	"sync/atomic"
)

type CheckLazySingleton struct {
	data uint64
}

var checkLazySingleton *CheckLazySingleton
var checkOnce = sync.Once{}

func CheckLazySingletonInstance() *CheckLazySingleton {
	if checkLazySingleton == nil {
		checkOnce.Do(func() {
			checkLazySingleton = &CheckLazySingleton{}
		})
	}

	return checkLazySingleton
}
```
sync.Once 使用原子操作来模拟锁的效果，来判断实例是否已经创建，和 LockCheckLazySingleton 中直接判断实例相比，并没有太多的性能开销，反而还可以避免并发时更多的运行实例进入等锁的状态，这种方式理论上效率会比上面那种方式更高一点。

## 性能测试
我们可以编写一个 benchmark 测试一下这几种单例模式实现方式的效率怎么样：

```go
func BenchmarkHungrySingletonInstance(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			if singleton.HungrySingletonInstance() != singleton.HungrySingletonInstance() {
				b.Errorf("different instance")
			}
		}
	})
}

func BenchmarkLazySingletonInstance(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			if singleton.LazySingletonInstance() != singleton.LazySingletonInstance() {
				b.Errorf("different instance")
			}
		}
	})
}

func BenchmarkLockCheckLazySingletonInstance(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			if singleton.LockCheckLazySingletonInstance() != singleton.LockCheckLazySingletonInstance() {
				b.Errorf("different instance")
			}
		}
	})
}

func BenchmarkCheckLazySingletonInstance(b *testing.B) {
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			if singleton.CheckLazySingletonInstance() != singleton.CheckLazySingletonInstance() {
				b.Errorf("different instance")
			}
		}
	})
}
```
输出结果如下所示：
```powershell
D:\git\design-pattern\singleton>go test -bench=.
goos: windows
goarch: amd64
pkg: design-pattern/singleton
cpu: Intel(R) Core(TM) i5-7200U CPU @ 2.50GHz
BenchmarkHungrySingletonInstance-4              1000000000               0.5729 ns/op
BenchmarkLazySingletonInstance-4                17065185                71.58 ns/op
BenchmarkLockCheckLazySingletonInstance-4       377396236                2.878 ns/op
BenchmarkCheckLazySingletonInstance-4           536262001                2.330 ns/op
PASS
ok      design-pattern/singleton        4.733s

```
我们可以看到，饿汉模式的效率是最高的，因为它的实例是在程序启动的时候就已经初始化好了，调用实例的过程省去了很多判断的过程。而直接加锁的懒汉模式是效率最低的，锁的开销是十分大的，实际使用中，如果有需要使用懒汉式的单例模式，也不推荐这种实现方式。后面两种优化的懒汉模式中，使用 once 的效率会更高一点，这种实现方式更加值得推荐。

## 单例模式的问题及替代方案

### 单例模式的问题

1. 单例对 OOP 特性（封装、继承、多态、抽象）的支持不友好。单例模式违反了基于接口而非实现的设计原则。
2. 单例会隐藏类之间的依赖关系。单例类不需要显示创建、不需要依赖参数传递，在代码比较复杂的情况下，调用关系就会非常隐蔽。
3. 单例对代码的扩展性不友好，在某些情况下会影响代码的扩展性、灵活性。
4. 单例对代码的可测试性不友好。单例类持有的变量通常是全局变量，被所有的代码共享，这不方便编写单元测试。
5. 单例不支持有参数的构造函数，因为单例类的对象只初始化一次，即使构造函数有参数也只接收第一次的参数。

### 替代方案

可以通过工厂模式、IOC容器（JAVA）等方式
