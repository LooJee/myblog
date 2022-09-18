---
title: '读源码:redigo为什么多线程不安全'
date: 2019-06-16 21:05:45
tags:
  - golang
url: why_redis_unsafe_in_concurrence.html
---

redigo是golang的一个操作redis的第三方库，之所以选择这个库，是因为它的文档十分丰富，操作起来也比较简单。一个典型的redigo的使用如下所示：

```golang
package main

import (
	"github.com/gomodule/redigo/redis"
	"log"
)

func main() {
	conn, err := redis.Dial("tcp", "192.168.1.2:6379")
	if err != nil {
		log.Fatalf("dial redis failed :%v\n", err)
	}

	result, err := redis.String(conn.Do("SET", "hello", "world"))
	if err != nil {
		log.Fatalln(err)
	}

	log.Println(result)
}
```

这里需要注意的一点是，redis 默认是只能本机访问的，可以通过修改 /etc/redis/redis.conf 中的 bind 来实现远程访问，这里我将 bind 改为了服务所在机器的 IP 。

虽然，redigo 的使用十分简单，但是，在它的文档中也指出了一点需要我们特别注意，我们可以在 [godoc](https://godoc.org/github.com/gomodule/redigo/redis#hdr-Concurrency) 中看到原文：

> Connections support one concurrent caller to the Receive method and one concurrent caller to the Send and Flush methods. No other concurrency is supported including concurrent calls to the Do and Close methods.

翻译过来就是：

> 连接支持同时运行单个执行体调用 Receive 和 单个执行体调用 Send 和 Flush 方法。不支持并发调用 Do 和 Close 方法。

本着程序员追根究底的好奇心，我看了一下 redigo 实现 Do 方法的源码，大致弄清楚了为什么 Do 函数是并发不安全的了。它的部分源码如下所示：

```golang
func (c *conn) Do(cmd string, args ...interface{}) (interface{}, error) {
	return c.DoWithTimeout(c.readTimeout, cmd, args...)
}

func (c *conn) DoWithTimeout(readTimeout time.Duration, cmd string, args ...interface{}) (interface{}, error) {
	c.mu.Lock()
	pending := c.pending
	c.pending = 0
	c.mu.Unlock()

	if cmd == "" && pending == 0 {
		return nil, nil
	}

	if c.writeTimeout != 0 {
		c.conn.SetWriteDeadline(time.Now().Add(c.writeTimeout))
	}

	if cmd != "" {
		if err := c.writeCommand(cmd, args); err != nil {
			return nil, c.fatal(err)
		}
	}

	if err := c.bw.Flush(); err != nil {
		return nil, c.fatal(err)
	}

	var deadline time.Time
	if readTimeout != 0 {
		deadline = time.Now().Add(readTimeout)
	}
	c.conn.SetReadDeadline(deadline)

	if cmd == "" {
		reply := make([]interface{}, pending)
		for i := range reply {
			r, e := c.readReply()
			if e != nil {
				return nil, c.fatal(e)
			}
			reply[i] = r
		}
		return reply, nil
	}

	var err error
	var reply interface{}
	for i := 0; i <= pending; i++ {
		var e error
		if reply, e = c.readReply(); e != nil {
			return nil, c.fatal(e)
		}
		if e, ok := reply.(Error); ok && err == nil {
			err = e
		}
	}
	return reply, err
}

func (c *conn) writeCommand(cmd string, args []interface{}) error {
	c.writeLen('*', 1+len(args))
	if err := c.writeString(cmd); err != nil {
		return err
	}
	for _, arg := range args {
		if err := c.writeArg(arg, true); err != nil {
			return err
		}
	}
	return nil
}
```

上面三个函数实现在 redigo 的 redis 包的 conn.go 文件中，在 DoWithTimeout 方法中，我们可以看到它是顺序执行数据的发送和相应的接收的，而且，函数中还是没有加锁的。虽然，golang 的 TCP 发送底层实现是有加锁的，可以保证一次写操作的数据中，不会有另一次写操作的数据插入，但是，在这个 DoWithTimeout 的实现中，我们还是能隐约闻到一种不安全的味道。

我们把焦点锁定在 writeCommand 这个方法上。从它的实现，我们可以了解到，它的作用主要是在 for ... range 中将 redis 的命令发送到 redis-server 执行。这时，我们可能会注意到，这个函数是没有加锁的，如果 for ... range 是往一个全局的缓冲去中写数据，那么，并发时很有可能会导致数据的交叉。为了证实这个假设，我们继续看 writeArg 的实现:

```golang
func (c *conn) writeArg(arg interface{}, argumentTypeOK bool) (err error) {
	switch arg := arg.(type) {
	case string:
		return c.writeString(arg)
	case []byte:
		return c.writeBytes(arg)
	case int:
		return c.writeInt64(int64(arg))
	case int64:
		return c.writeInt64(arg)
	case float64:
		return c.writeFloat64(arg)
	case bool:
		if arg {
			return c.writeString("1")
		} else {
			return c.writeString("0")
		}
	case nil:
		return c.writeString("")
	case Argument:
		if argumentTypeOK {
			return c.writeArg(arg.RedisArg(), false)
		}
		// See comment in default clause below.
		var buf bytes.Buffer
		fmt.Fprint(&buf, arg)
		return c.writeBytes(buf.Bytes())
	default:
		// This default clause is intended to handle builtin numeric types.
		// The function should return an error for other types, but this is not
		// done for compatibility with previous versions of the package.
		var buf bytes.Buffer
		fmt.Fprint(&buf, arg)
		return c.writeBytes(buf.Bytes())
	}
}

func (c *conn) writeString(s string) error {
	c.writeLen('$', len(s))
	c.bw.WriteString(s)
	_, err := c.bw.WriteString("\r\n")
	return err
}
```

writeArg 方法是通过判断传入参数的不同来调用不同的方法来写数据的，不过这几个方法的底层其实都是调用了 writeString 这个方法。在 writeString 这个方法的实现中，我们看到 redigo 是把数据都写到 bw 的。bw 是 conn 一个 net.Conn 的 writter，也就是说，如果并发执行 Do 方法的话，这几个并发的执行体都是往同一个 net.Conn的 writter 中写数据的，这基本证实了我上面的假设。

我们回过来看 DoWithTimeout 函数执行了 writeCommand 之后，调用的 bw 的 Flush 方法，这个方法将缓冲区中的数据都发送出去，我们看一下它的实现：

```golang
// Flush writes any buffered data to the underlying io.Writer.
func (b *Writer) Flush() error {
	if b.err != nil {
		return b.err
	}
	if b.n == 0 {
		return nil
	}
	n, err := b.wr.Write(b.buf[0:b.n])
	if n < b.n && err == nil {
		err = io.ErrShortWrite
	}
	if err != nil {
		if n > 0 && n < b.n {
			copy(b.buf[0:b.n-n], b.buf[n:b.n])
		}
		b.n -= n
		b.err = err
		return err
	}
	b.n = 0
	return nil
}
```

从代码中，我们可以看到，在调用了 b.wr.Write 方法后，有一个判断已写的数据长度是否和缓冲区的数据长度相等的操作。从上面的分析我们可以知道，redigo 在调用 Do 的整个过程中都是没有加锁的，那么，在并发时，一个执行体的 Flush 过程中，很有可能会有别的执行体往 writer 的缓冲区中写数据，出现在调用完 b.wr.Write 之后对已写数据长度小于缓冲区数据长度的现象，从而导致 short write 的错误。

我们可以写一个程序测试一下:

```golang
package main

import (
	"github.com/gomodule/redigo/redis"
	"log"
	"sync"
)

func main() {
	conn, err := redis.Dial("tcp", "192.168.1.2:6379")
	if err != nil {
		log.Fatalf("dial redis failed :%v\n", err)
	}

	wg := sync.WaitGroup{}
	wg.Add(2)

	go func() {
		defer wg.Done()
		result, err := redis.String(conn.Do("SET", "hello", "world"))
		if err != nil {
			log.Fatalln(err)
		}
		log.Println(result)
	}()

	go func() {
		defer wg.Done()
		result, err := redis.String(conn.Do("SET", "hello", "world"))
		if err != nil {
			log.Fatalln(err)
		}
		log.Println(result)
	}()

	wg.Wait()
}
```

执行之后，果然出现了 short write 的错误：

![](https://raw.githubusercontent.com/LooJee/medias/master/images/20190617235428.png)

redigo 的作者推荐我们在并发时使用连接池来保证安全，redigo 的连接池的实现将会在下次一起阅读。

读源码可以了解到开源作者实现开源作品的思路，还可以开拓视野，认识到一些更好的编程技巧，这个习惯可是要好好坚持啊。

