---
title: '读源码:redigo 的连接池'
date: 2019-06-23 23:20:48
tags:
  - golang
url: redigo_pool.html
---

[前面](https://tech-seeker.cn/2019/06/16/%E8%AF%BB%E6%BA%90%E7%A0%81-redigo%E4%B8%BA%E4%BB%80%E4%B9%88%E5%A4%9A%E7%BA%BF%E7%A8%8B%E4%B8%8D%E5%AE%89%E5%85%A8/)有说到，redigo 不是一个并发安全的 redis 库，它推荐在并发时使用连接池来访问 redis 服务器，redigo 连接池的典型使用方法如下所示：

```golang
package main

import (
	"github.com/gomodule/redigo/redis"
	"log"
)

func main() {
	pool := redis.Pool{
		Dial: func() (conn redis.Conn, e error) {
			return redis.Dial("tcp","192.168.1.10:6379")
		},
		MaxIdle:2,
	}
	defer pool.Close()

	conn := pool.Get()
	reply, err := redis.String(conn.Do("SET", "hello", "world"))
	if err != nil {
		log.Println(err)
	}

	log.Println(reply)
	conn.Close()
}
```

## Pool 结构说明

在介绍 pool 几个重要方法的实现之前，我们先来看一下 redis.Pool 结构的一些参数，godoc 的传送门在[这里](https://godoc.org/github.com/gomodule/redigo/redis#Pool)。

| 参数名          | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| Dial            | 该参数为链接redis的函数，每次从连接池中获取连接时，如果没有空闲的链接，就将会用该函数创建新的链接 |
| TestOnBorrow    | 测试连接状态的函数                                           |
| MaxIdle         | 连接池最大可有的空闲连接数                                   |
| MaxActive       | 连接池最大可有的连接数，这个参数通常和下面的Wait参数同时使用 |
| IdleTimeout     | 空闲连接的超时时间                                           |
| Wait            | 当该值为True，并且设置了MaxActive，当已使用的连接数已经达到了MaxActive，那么，Get函数会一直等待，直到有连接可用；如果设置了MaxActive，但是Wait为False，那么，当没有连接可用时，会直接返回 [ErrPoolExhausted](https://godoc.org/github.com/gomodule/redigo/redis#ErrPoolExhausted) 的错误。 |
| MaxConnLifetime | 连接的生命周期，当调用Get时，会判断连接是否超时，如果超时，会将连接关闭 |
| chInitialized   | 该值用来判断下面的ch参数是否已经初始化                       |
| mu              | 互斥锁就不需要介绍了                                         |
| closed          | 判断连接池是否已经关闭                                       |
| active          | 记录连接池中的活跃连接数，该值和下面的idle可以通过方法 [Stats](https://godoc.org/github.com/gomodule/redigo/redis#Pool.Stats)() 来查看 |
| ch              | 该值和Wait参数配合使用，它会被初始化成缓冲区长度和MaxIdle相同的channel，每当使用一个连接时，就会在ch中写入一个值，当使用的连接数和MaxIdle相等时，ch就会阻塞，直到有连接回收 |
| idle            | 连接池中空闲的连接数，可以通过方法 [Stats](https://godoc.org/github.com/gomodule/redigo/redis#Pool.Stats)() 来查看。它是一个链表，它的结构是这样的：type idleList struct {count int; front, back *poolConn;} |

## Get方法

当新建一个连接池之后，我们使用方法 Get 从连接池中取出一个连接来进行相关的操作，Get 方法的实现如下所示：

```golang
func (p *Pool) Get() Conn {
	pc, err := p.get(nil)
	if err != nil {
		return errorConn{err}
	}
	return &activeConn{p: p, pc: pc}
}
```

如果能成功通过 get 方法获取连接，则返回 activeConn 的实例，否则，返回 errConn 的实例。如果，返回的是 errConn 的实例的话，那么，不论调用什么方法都会直接返回错误 err，十分巧妙的设计。

现在来看一下 get 方法的实现。get 方法的第一段代码如下所示：

```golang
if p.Wait && p.MaxActive > 0 {
	p.lazyInit()
	if ctx == nil {
		<-p.ch
	} else {
		select {
		case <-p.ch:
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
}
```

这段代码只有在 p.Wait 为 True 并且设置了 MaxActive  的时候才会执行，它会在 lazyInit（实现如下）中初始化 p.ch ，并且往 p.ch 中写满数据，这样就可以保证能获取 MaxActive 个连接，多余的 Get 请求将会阻塞在 <-p.ch，直到 p.ch 中写入数据，即有连接空闲。

```golang
unc (p *Pool) lazyInit() {
	// Fast path.
	if atomic.LoadUint32(&p.chInitialized) == 1 {
		return
	}
	// Slow path.
	p.mu.Lock()
	if p.chInitialized == 0 {
		p.ch = make(chan struct{}, p.MaxActive)
		if p.closed {
			close(p.ch)
		} else {
			for i := 0; i < p.MaxActive; i++ {
				p.ch <- struct{}{}
			}
		}
		atomic.StoreUint32(&p.chInitialized, 1)
	}
	p.mu.Unlock()
}
```

get 方法中第二段代码是用来判断连接池中是否有超时的空闲连接，先来看一下代码：

```golang
if p.IdleTimeout > 0 {
	n := p.idle.count
	for i := 0; i < n && p.idle.back != nil && p.idle.back.t.Add(p.IdleTimeout).Before(nowFunc()); i++ {
		pc := p.idle.back
		p.idle.popBack()
		p.mu.Unlock()
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}
}
```

因为，redigo 连接池的空闲连接链表 idle 是一个先进后出的链表，所以，back 所指向的空闲连接是相对较老的，所以，这里是从 idle.back 开始判断是否已经超时，如果超时，则将其从链表中取出，并且关闭该连接。

第三段代码是从 idle 中取出空闲的连接：

```golang
for p.idle.front != nil {
	pc := p.idle.front
	p.idle.popFront()
	p.mu.Unlock()
	if (p.TestOnBorrow == nil || p.TestOnBorrow(pc.c, pc.t) == nil) &&
		(p.MaxConnLifetime == 0 || nowFunc().Sub(pc.created) < p.MaxConnLifetime) {
		return pc, nil
	}
	pc.c.Close()
	p.mu.Lock()
	p.active--
}
```

如上所说，它是从 idle 的 front 取出一个空闲的连接。如果有设置测试函数，会先执行测试；如果有设置MaxConnLifetime，那么将会判断连接的生命周期是否已经超过该值。如果能在 idle 中取得符合条件的连接，那么会将其返回，否则会将不符合的连接关闭直到取出符合条件的连接，或者直到 idle 中没有空闲的连接，那么将会继续往下执行。

再往下是两个判断：

```golang
if p.closed {
	p.mu.Unlock()
	return nil, errors.New("redigo: get on closed pool")
}

if !p.Wait && p.MaxActive > 0 && p.active >= p.MaxActive {
	p.mu.Unlock()
	return nil, ErrPoolExhausted
}
```

第一个是判断连接池是否已经关闭。第二个是判断未设置 Wait 但是设置了 MaxActive 的情况下，已创建的连接数是否已经达到了 MaxActive，如果条件满足，则返回错误。

最后一段代码是创建并返回连接：

```golang
p.active++
p.mu.Unlock()
c, err := p.Dial()
if err != nil {
	c = nil
	p.mu.Lock()
	p.active--
	if p.ch != nil && !p.closed {
		p.ch <- struct{}{}
	}
	p.mu.Unlock()
}
return &poolConn{c: c, created: nowFunc()}, err
```

如果创建失败，那么，它会重新往 p.ch 中写入值。因为在第一段代码中从 p.ch 取出了一个值，如果创建失败了不重新写入，那么等于 p.ch 中能缓冲的数据就减少了，从而导致能从连接池中获取的连接就减少了，最后可能就在第一段代码那里死锁了。

## 连接的回收

在一个不使用连接池的场景中，当调用了 conn.Close() 之后，当前的连接将会被关闭，而在使用连接池的情况下，调用 conn.Close() 会发生什么呢？我们来看一下连接池返回的连接，也就是 activeConn 的 Close 方法的实现：

```golang
func (ac *activeConn) Close() error {
	pc := ac.pc
	if pc == nil {
		return nil
	}
	ac.pc = nil

	if ac.state&internal.MultiState != 0 {
		pc.c.Send("DISCARD")
		ac.state &^= (internal.MultiState | internal.WatchState)
	} else if ac.state&internal.WatchState != 0 {
		pc.c.Send("UNWATCH")
		ac.state &^= internal.WatchState
	}
	if ac.state&internal.SubscribeState != 0 {
		pc.c.Send("UNSUBSCRIBE")
		pc.c.Send("PUNSUBSCRIBE")
		// To detect the end of the message stream, ask the server to echo
		// a sentinel value and read until we see that value.
		sentinelOnce.Do(initSentinel)
		pc.c.Send("ECHO", sentinel)
		pc.c.Flush()
		for {
			p, err := pc.c.Receive()
			if err != nil {
				break
			}
			if p, ok := p.([]byte); ok && bytes.Equal(p, sentinel) {
				ac.state &^= internal.SubscribeState
				break
			}
		}
	}
	pc.c.Do("")
	ac.p.put(pc, ac.state != 0 || pc.c.Err() != nil)
	return nil
}
```

前面一部分代码都是和 redis 服务器进行通信，释放相关的资源，这里主要看 put 方法的实现：

```golang
func (p *Pool) put(pc *poolConn, forceClose bool) error {
   p.mu.Lock()
   if !p.closed && !forceClose {
      pc.t = nowFunc()
      p.idle.pushFront(pc)
      if p.idle.count > p.MaxIdle {
         pc = p.idle.back
         p.idle.popBack()
      } else {
         pc = nil
      }
   }

   if pc != nil {
      p.mu.Unlock()
      pc.c.Close()
      p.mu.Lock()
      p.active--
   }

   if p.ch != nil && !p.closed {
      p.ch <- struct{}{}
   }
   p.mu.Unlock()
   return nil
}
```

首先，它将需要释放的连接插入了 idle 的头部。如果此时空闲的连接数超过了 MaxIdle，那么，将会把尾部的空闲连接即最老的空闲连接取出，并将其关闭。最后，往 p.ch 中写入值，保证在设置了 Wait 的情况下正常运行。

## 结语

redigo 的连接池就大概分享完了，时间比较匆忙，感觉总结的不是特别好，希望大家能多多斧正。