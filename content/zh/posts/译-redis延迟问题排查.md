---
title: 译-redis延迟问题排查
date: 2019-09-15 19:21:48
tags:
  - redis
  - 学习笔记
---

> 本文是对 redis 官方文档 [Redis latency problems troubleshooting](https://redis.io/topics/latency) 的翻译，诸位看官可结合原文食用。

这个文档将会帮助你理解在使用 Redis 遇到延迟问题时发生了什么。

在这个上下文中，*延迟*指的是从客户端发送命令和接收到应答之间所消耗的时间。通常情况下，Redis 处理命令的时间十分短，在亚微妙之间，但是某些情况会导致高延迟。

## 我很忙，给我清单

接下来的内容对于以低延迟方式运行 Redis 是非常重要的。然而，我理解我们都是很忙的，所以就以快速清单开始吧。如果你在尝试了下面这些步骤失败了，请回来阅读完整的文档。

1. 确认你运行慢查询命令阻塞了服务。使用 Redis [慢日志功能](https://redis.io/commands/slowlog)确认这一点。
2. 对于 EC2 用户，确认你在基于现代的 EC2 实例上使用 HVM，比如 m3.medium。否则 fork() 会非常慢。
3. 内核的 **Transparent huge pages** 必须请用。使用 echo never > /sys/kernel/mm/transparent_hugepage/enabled 来禁用他们，然后重启 Redis 进程。
4. 如果你使用虚拟机，那么即使你没有对 Redis 做任何操作也可能会有内部延迟。使用 ./redis-cli --intrinsic-latency 100 来在你的运行环境中检查最小延迟。注意：你需要在服务端而不是客户端运行这个命令。
5. 启用 Redis 的 [Latency monitor](https://redis.io/topics/latency-monitor) 功能，这是为了得到你的 Redis 实例中的延迟事件和原因的可读数据。

通常，使用下表来进行持久化和延迟/性能之间的权衡，从更好的安全性到更好的延迟排序。

1. AOF + fsync always：这会非常慢，你应该只在你知道自己在做什么的情况下使用。
2. AOF + fsync every second：这会是一个很好的折衷方案。
3. AOF + fsync every second + no-appendfsync-on-rewrite 选项设置为 yes：这也是一个折衷方案，但它避免了在重写时执行 fsync ，降低了磁盘压力。
4. AOF + fsync never：这个方案将 fsync 交给了内核，得到了更小的磁盘压力和延迟峰值风险。
5. RDB：这个方案你将会有更广泛的权衡空间，它取决于你如何触发 RDB 持久化。

现在，我们花 15 分钟来了解下细节...

## 测量延迟

如果你遇到了延迟问题，你可能知道如何在你的程序中测量延迟，或者延迟现象十分明显。其实，redis-cli 可以在毫秒级内测量 Redis 服务的延迟，只要使用下面的命令：

```
redis-cli --latency -h `host` -p `port`
```

## 使用 Redis 内置的延迟监控子系统

从 Redis 2.8.13 开始，Redis 提供了延迟监控功能，它通过对不同执行路径（译者注：这个概念可以在延迟监控文档查看）采样来检测服务阻塞的位置。这将会使调试本文档所说明的问题更加方便，所以我们建议尽快启用延迟监控。请查看[延迟监控文档](https://redis.io/topics/latency-monitor)。

延迟监控采样和报告功能会使你更方便的找出 Redis 系统延迟的原因，不过我们还是建议你详细阅读本文以更好的理解 Redis 和延迟尖峰。

## 延迟基准线

有一种延迟本身就是你运行 Redis 的环境的一部分，即操作系统内核以及--如果你使用了虚拟化--管理程序造成的延迟。

这种延迟无法消除，学习它是十分重要的，因为它是基准线，换句话说，因为内核和管理程序的存在，你不可能使 Redis 的延迟比在你的环境中运行的程序的延迟更低。

我们将这种延迟称为内部延迟，redis-cli 从 Redis 2.8.7 开始就可以测量它了。这是一个在基于 Linux 3.11.0 的入门级服务器上运行的例子。

注意：参数 100 是测试执行的秒数。执行测试的时间越长，我们越有可能发现延迟峰值。100 秒通常足够了，但是你可能想要运行多次不同时长的测试。请注意这个测试是 CPU 密集型的，将可能会使你的系统的一个核满载。

![](https://raw.githubusercontent.com/LooJee/medias/master/images/20190915215506.png)

注意： redis-cli 在这个例子中需要运行在你运行 Redis 或者计划运行 Redis 的服务器上，而不是在客户端。在这个特殊模式下 redis-cli 根本不会连接 Redis 服务：它只是尝试测试内核不提供 CPU 时间去运行 redis-cli 进程自己的最大时长。

在上面的例子中，系统的内部延迟只是 0.115毫秒（或 115 微妙），这是一个好的消息，但是请记住，内部延迟可能会随运行时间变化和变化，这取决与系统的负载。

虚拟化环境表现将会差点，特别是当高负载或者受其他虚拟化环境影响。下面是在 Linode 4096 实例上运行 Redis 和 Apache 的结果：

![](https://raw.githubusercontent.com/LooJee/medias/master/images/20190915220737.png)

这里我们有 9.7 毫秒的内部延迟：这意味着我们不能要求 Redis 做的比这个更好了。然而，在负载更高或有其它邻居的不同虚拟化环境中运行时，可以更容易得到更差的结果。我们可能在正常运行的系统中得到 40 毫秒的测试结果。

## 受网络和通讯影响的延迟

客户端通过 TCP/IP 或者 Unix 域来连接到 Redis。1 Gbit/s 的网络的典型延迟是 200 微妙，使用 Unix 域 socket 的延迟可能低至 30 微妙。它实际上取决于你的网络和系统硬件。在通讯之上，系统添加了更多的延迟（由于线程调度，CPU 缓存，NUMA 配置，等等）。系统在虚拟环境中造成的延迟明显高于物理机。

结论是，即使 Redis 处理大部分命令只花费亚微秒级的时间，客户端和服务端之间大量往返的命令将必须为网络和系统相关延迟付出代价。

因此，高效的客户端会将多个命令组成流水线来减少往返的次数。这被服务器和大多数客户端支持。批量操作比如 MSET/MGET 也是为了这个目的。从 Redis 2.4 开始，一些命令还支持所有数据类型的可变参数。

这里是一些准则：

1. 如果负担得起，使用物理机而不是 VM 来托管服务。
2. 不要总是连接和断开连接（特不是基于 web 的应用）。尽可能地保持连接。
3. 如果你的客户端和服务端在同一台机器上，使用 Unix 域套接字。
4. 使用批量命令（MSET/MGET），或使用带可变参数的命令，而不是使用流水线操作。
5. 比起一系列的单独命令，更应该选择流水线操作（如果可能）。
6.  Redis 支持 Lua服务器端脚本，以涵盖不适合原始流水线操作的情况（例如，当命令的结果是下一个命令的输入时）。

在 Linux，用户可以通过进程设置（taskset），cgroups，实时优先级（chrt），NUMA 设置，或者使用低延迟内核等来实现更好的延迟。请注意 Redis 并不适合绑定在单个 CPU 核心上。Redis 会 fork 后台任务像 bgsave 或 AOF，这操作会十分消耗 CPU。这些任务必须永远不和主进程在同一个核心上运行。

在大多环境中，这些系统级的优化不是必须的。只有当你需要它们或者熟悉它们的时候再去做这些操作。

## Redis 的单线程特性

Redis 被设计为大部分情况下使用单线程。这意味着一个线程处理所有客户端的请求，它使用了多路复用的技术。这意味着 Redis 在每个是简单都只处理一个请求，所以所有的请求都是按顺序处理的。这很像 Node.js 的工作方式。然而，这两个产品通常都不会被认为很慢。这是因为他们处理任务的时间很短，不过主要是因为它们设计为不在系统调用阻塞，特别是从套接字读取数据或者往套接字写数据的时候。

我说 Redis 大多只用单线程，是因为从 Redis 2.4 开始，我们使用多线程去在后台执行一些慢 I/O 操作，主要是和磁盘 I/O 相关，但是这不改变 Redis 使用单线程处理所有请求的事实。

## 慢查询导致的延迟

一个单线程的后果是当有一个慢请求时，所有其它的客户端将会等待它完成。当执行像 GET 或 SET 或 LPUSH 这样的通常命令时是没有问题的，这些命令的执行时间都是常数的（非常短）。然而，有几个命令操作了大量的元素，像 SORT，LREM，SUNION 和其它的命令。例如，去两个大集合的交集会花费非常多的时间。

所有命令的算法复杂度都有文档记录。一个好的实践是，当你使用不熟悉的命令时先系统地测试一下它。

如果你有延迟的顾虑，那么你不应该用慢查询处理有大量元素的值，或者你应该运行 Redis 的副本去运行慢查询。

可以使用 Redis 的[慢日志功能](https://redis.io/commands/slowlog)来监控慢查询。

另外，你可以使用你喜欢的进程监控程序（top，htop，prstat，等等）去快速的检查主 Redis 进程 所消耗的 CPU。如果它很高，但流量却不高，那么它通常表示正在执行慢查询。

重点：一个非常常见的由执行慢查询导致延迟的原因是在生产环境执行 KEYS 命令。KEYS 命令在文档中指出只能用于调试目的。从 Redis 2.8 开始，引入了一些新的命令来迭代键空间和其它大集合，请查看 [SCAN](https://redis.io/commands/scan)，[SSCAN](https://redis.io/commands/sscan)，[HSCAN](https://redis.io/commands/hscan) 和 [ZSCAN](https://redis.io/commands/zscan) 命令来获取更多的信息。

## 由 fork 引发的延迟

为了在后台生成 RDB 文件或者在启用 AOF 持久化时重写 AOF 文件，Redis 必须 fork 一个进程。fork 操作（在主线程运行）会引起延迟。fork 是在类 Unix 系统中一个开销很高的操作，因为它涉及到复制大量与进程关联的对象。对于和虚拟内存相关联的页表尤其如此。

例如，在 Linux/AMD64 系统，内存被分为每页 4 kB。为了将虚拟地址转换为物理地址，每个进程保存了一个页表（实际是一棵树），它至少包含一个指向进程的每页地址空间的指针。所以，一个拥有 24 GB 的 Redis 实例需要 24 GB/4 KB*8 = 48 MB 的页表。

当 bgsave 执行的时候，实例将会被 fork，这将创建和拷贝 48 MB 的内存。它会花费时间和 CPU，特别是在虚拟机上创建和初始化大页面将会是非常大的开销。

## 不同系统上 fork 的时间

现代硬件拷贝页表非常快，除了 Xen。这个问题不是出在 Xen 虚拟化，而是 Xen 本身。例如，使用 VMware 或 Virtual Box 不会导致缓慢的 fork 时间。下表是比较不同的 Redis 实例 fork 所消耗的时间。数据来自于执行BGSAVE，并观察INFO命令输出的`latest_fork_usec`信息。

然而，好消息是基于 EC2 HVM 的实例执行 fork 操作的表现很好，几乎和在物理机上执行差不多，所以使用 m3.medium （或高性能）的实例将会得到更好的结果。

1. 运行在 VMware 上的虚拟 Linux 系统 fork 6.0 GB 的 RSS 花费 77毫秒（每 GB 12.8 毫秒）。
2. Linux 运行在物理机上（未知硬件）fork 6.1 GB 的 RSS 花费了 80 毫秒（每 GB 13.1 毫秒）。
3. Linux 运行在物理机上（Xeon @ 2.27 Ghz）fork 6.9 GB 的 RSS 花费 62 毫秒（每 GB 9 毫秒）。
4. 运行在 6sync 上的 Linux 虚拟机（KVM）fork 360 MB 的 RSS 花费 8.2 毫秒（每 GB 23.3 毫秒）。
5. 运行在老版本 EC2 上的 Linux 虚拟机（Xen）fork 6.1 GB 的 RSS 花费 1460 毫秒（每 GB 10 毫秒）。
6. 运行在新版本 EC2 上的 Linux 虚拟机（Xen）fork 1 GB 的 RSS 花费 10 毫秒（每 GB 10 毫秒）。
7. 运行在 Linode 上的 Linux 虚拟机（Xen）fork 0.9 GB 的 RSS 花费 382 毫秒（每 GB 424 毫秒）。

正如您所看到的，在 Xen 上运行的某些虚拟机的性能损失介于一个数量级到两个数量级之间。对于 EC2 用户，建议很简单：使用基于 HVM 的现代实例。

## transparent huge pages引起的响应延迟

很不幸，如果 Linux 内核开启了 transparent huge pages 功能，Redis 将会在调用 fork 来持久化到磁盘时造成非常大的延迟。大内存页导致了下面这些问题：

1. 当调用fork时，共享大内存页的两个进程将被创建。
2. 在一个忙碌的实例上，一些事件循环就将导致访问上千个内存页，导致几乎整个进程执行写时复制。
3. 这将导致高响应延迟和大内存的使用。

请确认使用以下命令禁用 transparent huge pages：

```shell
echo never > /sys/kernel/mm/transparent_hugepage/enabled
```

## 页面交换（操作系统分页）引起的延迟

Linux（还有很多其它现代操作系统）可以将内存页从内存缓存到磁盘，或从磁盘读入内存，这是为了更有效的使用系统内存。

如果 Redis 页被内核从内存保存到了交换文件，当保存在这个内存页中的数据被 Redis 使用到的时候（比如访问保存在这个页中的键）内核为了将这页移动到主存，会停止 Redis 进程。访问随机 I/O 是一个缓慢的操作（和访问在内存中的页相比）而且 Redis 客户端将会经历异常的延迟。

内核将 Redis 内存页保存到磁盘主要有三个原因：

1. 系统正在内存压力之下，因为运行的进程要求了比可用内存更多的物理内存。最简单的例子是 Redis 使用了比可用内存更多的内存。
2. Redis 实例的数据集合或一部分数据集合已经基本完全空闲（从未被客户端访问），所以内核可能会将空闲的内存页保存到磁盘。这个问题十分罕见，因为即使是一个不是特别慢的实例通常也会访问所有的内存页，强迫内核将所有的页保存在内存中。
3. 系统中的一些进程引发了大量读写 I/O 的操作。因为文件会产生缓存，它会给内核增加文件系统缓存的压力，因此就会产生交换活动。请注意，这包括 Redis RDB 和/或 AOF 线程，它们会产生大文件。

幸运的是 Linux 提供了好的工具去验证这个问题，所以，最简单的事是当怀疑是交换内存引起延迟时，那就去检查它吧。

第一件事要做的是检查 Redis 内存交换到磁盘的数量。这之前，你需要获取 Redis 实例的 pid：

```shell
$ redis-cli info | grep process_id
process_id:5454
```

现在进入 /proc 目录下该进程的目录：

```shell
$ cd /proc/5454
```

你将会在这里找到一个叫做 smaps 的文件，它描述了 Redis 进程的内存布局（假设你使用的是 Linux 2.6.16 以上的系统）。这个文件包含了和我们的进程内存映射相关的非常详细的信息，一个名为 Swap 的字段就是我们所需要的。然而，当 smaps 文件包含了不同的关于 Redis 进程的内存映射之后，它就不是单单一个 swap 字段了（进程的内存布局比一页简单的线性表要远远复杂）。

因为我们对进程相关的内存交换都感兴趣，所以第一件事要做的是用 grep 得到文件中所有的 Swap 字段：

```shell
$ cat smaps | grep 'Swap:'
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                 12 kB
Swap:                156 kB
Swap:                  8 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  4 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  4 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  4 kB
Swap:                  4 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
Swap:                  0 kB
```

如果所有都是 0 KB，或者零散一个 4k 的条目，那么一切都很完美。事实上，在我们的示例中（一个运行 Redis 的网站，每秒服务百余用户）有些条目会显示更多的交换页。为了研究这是否是一个严重的问题，我们更改命令以便打印内存映射的大小：

```shell
$ cat smaps | egrep '^(Swap|Size)'
Size:                316 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
Size:                  8 kB
Swap:                  0 kB
Size:                 40 kB
Swap:                  0 kB
Size:                132 kB
Swap:                  0 kB
Size:             720896 kB
Swap:                 12 kB
Size:               4096 kB
Swap:                156 kB
Size:               4096 kB
Swap:                  8 kB
Size:               4096 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
Size:               1272 kB
Swap:                  0 kB
Size:                  8 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
Size:                 16 kB
Swap:                  0 kB
Size:                 84 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
Size:                  8 kB
Swap:                  4 kB
Size:                  8 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  4 kB
Size:                144 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  4 kB
Size:                 12 kB
Swap:                  4 kB
Size:                108 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
Size:                272 kB
Swap:                  0 kB
Size:                  4 kB
Swap:                  0 kB
```

正如你从输出所见，这里有一个映射有 720896 kB（只有 12 KB 被交换），在另一个映射中有 156 KB 被交换：基本上占我们内存非常少的部分被交换，所以不会造成大的问题。

相反，如果有大量进程内存页被交换到磁盘，那么你的响应延迟问题可能和内存页交换有关。如果是这样的话，你可以使用`vmstat`命令来进一步检查你的 Redis 实例：

![](https://raw.githubusercontent.com/LooJee/medias/master/images/20190917204835.png)

输出中我们需要的一部分是两列 `si` 和 `so` ，它们记录了内存从 swap 文件交换出和交换入的次数。如果你看到的这两列是非 0 的，那么你的系统上存在内存交换。

最后， iostat 命令可以用于检测系统的全局 I/O 活动。

![](https://raw.githubusercontent.com/LooJee/medias/master/images/20190917205312.png)

如果延迟问题是由于Redis内存在磁盘上交换造成的，则需要降低系统内存压力，如果Redis使用的内存超过可用内存，则增加更多内存，或者避免在同一系统中运行其他内存耗尽的进程。

## 由于AOF和磁盘I / O导致的延迟

另一个导致延迟的原因是 Redis 支持的 AOF 功能。AOF 基本上只使用两个系统调用来完成工作。一个是 `write(2)`，它用来追加数据到文件，另一个是 `fdatasync(2)` ，用来刷新内核文件缓存到磁盘，用来确保用户指定的持久化级别。

 `write(2)`和 `fdatasync(2)`都有可能导致延迟。例如，当正在进行系统大范围同步，或者输出缓冲区已满，内核需要将数据刷新到磁盘来保证接受新的写操作时，都有可能阻塞  `write(2)`。

`fdatasync(2)`调用是一个导致延迟的更糟糕的原因，因为使用的内核和文件系统的许多组合可能需要几毫秒到几秒才能完成，特别是在某些其他进程执行 I/O 的情况下。出于这个原因，Redis 2.4 可能会在不同的线程中执行`fdatasync(2)`调用。

我们将会看到如何设置可以改善使用 AOF 文件引起的延迟问题。

可以使用`appendfsync`配置选项将 AOF配 置为以三种不同方式在磁盘上执行 fsync（可以使用`CONFIG SET`命令在运行时修改此设置）。

1. 当 `appendfsync` 被设置为 `no`时，Redis 将不会执行 fsync。这种设置方式里，只有`write(2)`会导致延迟。这种情况下发生延迟通常没有解决方法，原因非常简单，因为磁盘拷贝的速度无法跟上 Redis 接收数据的速度，不过这种情况十分不常见，除非因为其他进程在操作 I/O 导致磁盘读写很慢。
2. 当 `appendfsync`被设置为 `everysec` 时，Redis 每秒执行一次 fsync。它会使用一个不同的线程，并且当 fsync 在运行的时候，Redis 会使用 buffer 来延迟 `write(2)` 的调用 2 秒左右（因为在 Linux 中，当 write 和 fsync 在进程中竞争同一个文件时会导致阻塞）。然而，如果 fsync 占用了太长的时间，Redis 最终会执行 write 调用，这将会导致延迟。
3. 当 `appendfsync` 被设置为 `always` 时，fsync 会在每次写操作时执行，这个操作发生在回复 OK 应答给客户端之前（事实上 Redis 会尝试将多个同时执行的命令集合到一个 fsync 中）。在这种模式下性能通常会非常慢，非常推荐使用快的磁盘和执行 fsync 很快的文件系统。

大多数 Redis 用户会使用 `no` 或者 `everysec` 来设置 `appendfsync`。获得最少的延迟的建议时避免其它进程在同一系统中操作 I/O。使用 SSD 硬盘可以提供很好的帮助，不过如果磁盘是空闲的，那么非 SSD 磁盘也能有很好的表现，因为 Redis 写 AOF 文件的时候不需要执行任何查找。

如果想要确认延迟是否和 AOF 文件相关，在 Linux 下你可以使用 `strace `命令来查看：

```shell
sudo strace -p $(pidof redis-server) -T -e trace=fdatasync
```

上面的命令将会显示 Redis 在主线程执行的所有 `fdatasync(2)` 命令。你不能通过上面的命令查看当 appendfsync 设置为 `everysec`时在后台线程执行的 `fdatasync`命令。可以添加 -f 参数来查看后台执行的 `fdatasync`命令。

如果你想要同时查看 `fdatasync` 和 `write` 两个系统调用，可以使用下面的命令:

```shell
sudo strace -p $(pidof redis-server) -T -e trace=fdatasync,write
```

不过，因为 `write`命令还被用于写数据到客户端的套接字，所以可能会显示很多和磁盘 I/O 无关的数据。显然没有办法告诉 strace 只显示慢速系统调用，所以我使用以下命令：

```shell
sudo strace -f -p $(pidof redis-server) -T -e trace=fdatasync,write 2>&1 | grep -v '0.0' | grep -v unfinished
```

## 到期产生的延迟

Redis 使用下面两种方法来处理过期的键：

1. 一种被动的方式是，在一个命令被请求时，如果发现它已经过期了，那就将它删除。
2. 一种主动的方式是，每 100 毫秒删除一些过期的键。

主动过期的方式被设计为自适应的。每 100 毫秒一个周期（每秒 10 次），它将进行下面的操作：

1. 采用`ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP` 键，删除所有已经过期的键。
2. 如果超过 25% 的键是已经过期的，重复这个过程。

`ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP` 的默认值是 20，这个过程每秒执行 10 次，通常没有将会有 200 个键因过期被主动删除。这已经能够快地清除 DB 了，即使有些键已经很久没有被访问了，所以被动算法是没什么帮助的。同时，每秒删除 200 个键并不会对 Redis 的延迟造成影响。

然而，这个算法是自适应的，如果在采样键集合中找到了超过  25% 的键是过期的，它会循环执行。但是运行这个算法的周期是每秒 10 次，这意味着可能发生在同一秒内被采样的键 25% 以上都过期的情况。

基本上，这意味着 **如果数据库在同一秒内有非常多键过期了，而且它们构成了当前键集合 25% 的失效键集合**，Redis 可能会阻塞着直到失效键的比例降到 25% 以下。

这种方式是为了避免因为失效的键使用过多的内存，而且通常都是无害的，因为在同一秒内出现大量失效的键是十分奇怪的，但是用户在同一 Unix 时间广泛使用 `EXPIREAT` 命令也不是不可能。

简而言之：注意大量键同时过期引起的响应延迟。

## Redis 软件看门狗

Redis 2.6 引入了 **Redis 软件看门狗**这个调试工具，它被设计用来跟踪使用其它通常工具无法分析的导致延迟的问题。

软件看门狗是一个试验性的特性。虽然它被设计为用于开发环境，不过在继续使用之前还是应该备份数据库，因为它可能会对 Redis 服务器的正常操作造成不可预知的影响。

它只有在通过其它方法没有办法找到原因的情况下才能使用。

下面是这个特性的工作方式：

1. 用户使用 `CONFIG SET` 命令启用软件看门狗。
2. Redis 开始一直监控自己。
3. 如果 Redis 检测到服务器阻塞在了某些操作中，没有即使返回，并且这可能是导致延迟的原因，一个低层的关于服务器阻塞位置的报告将会生成到日志文件中。
4. 用户在 Redis 的 Google Group 中联系开发者，并展示监控报告的内容。

注意，这个特性不能在 `redis.conf`文件中启用，因为它被设计为只能在已经运行的实例中启用，并且只能用于测试用途。

使用下面的命令来启用这个特性：

```shell
CONFIG SET watchdog-period 500
```

period 被指定为毫秒级。上面这个例子是指在检测到服务器有 500 毫秒或以上的延迟时则记录到日志文件中。最小可配置的延迟时间是 200 毫秒。

当你使用完了软件看门狗，你可以将`watchdog-period`参数设置为 0 来关闭这一功能。

**重点**：一定要记得关闭它，因为长时间开启看门狗不是一个好的注意。

下面的内容是当看门狗检测到延迟大于设置的值时会记录到日志文件中的内容：

```shell
[8547 | signal handler] (1333114359)
--- WATCHDOG TIMER EXPIRED ---
/lib/libc.so.6(nanosleep+0x2d) [0x7f16b5c2d39d]
/lib/libpthread.so.0(+0xf8f0) [0x7f16b5f158f0]
/lib/libc.so.6(nanosleep+0x2d) [0x7f16b5c2d39d]
/lib/libc.so.6(usleep+0x34) [0x7f16b5c62844]
./redis-server(debugCommand+0x3e1) [0x43ab41]
./redis-server(call+0x5d) [0x415a9d]
./redis-server(processCommand+0x375) [0x415fc5]
./redis-server(processInputBuffer+0x4f) [0x4203cf]
./redis-server(readQueryFromClient+0xa0) [0x4204e0]
./redis-server(aeProcessEvents+0x128) [0x411b48]
./redis-server(aeMain+0x2b) [0x411dbb]
./redis-server(main+0x2b6) [0x418556]
/lib/libc.so.6(__libc_start_main+0xfd) [0x7f16b5ba1c4d]
./redis-server() [0x411099]
------
```

注意：在例子中 **DEBUG SLEEP** 命令是用来组在服务器的。如果服务器阻塞在不同的位置，那么堆栈信息将会是不同的。

如果你碰巧收集了多份看门狗堆栈信息，我们鼓励你将他们都发送到 Redis 的 Google Group：我们收集到越多的堆栈信息，那么你的实例的问题将越容易被理解。

