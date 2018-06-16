# 5.6. Ratelimit 服务流量限制

计算机程序可依据其瓶颈分为 Disk IO-bound，CPU-bound，Network-bound，分布式场景下有时候也会外部系统而导致自身瓶颈。

web 系统打交道最多的是网络，无论是接受，解析用户请求，访问存储，还是把响应数据返回给用户，都是要走网络的。在没有 epoll/kqueue 之类的系统提供的 IO 多路复用接口之前，多个核心的现代计算机最头痛的是 C10k 问题，C10k 问题会导致计算机没有办法充分利用 CPU 来处理更多的用户连接，进而没有办法通过优化程序提升 CPU 利用率来处理更多的请求。

从 linux 实现了 epoll，freebsd 实现了 kqueue，这个问题基本解决了，我们可以借助内核提供的 API 轻松解决当年的 C10k 问题，也就是说如今如果你的程序主要是和网络打交道，那么瓶颈一定在用户程序而不在操作系统内核。

随着时代的发展，编程语言对这些系统调用又进一步进行了封装，如今做应用层开发，几乎不会在程序中看到 epoll 之类的字眼，大多数时候我们就只要聚焦在业务逻辑上就好。Golang 的网络库针对不同平台封装了不同的 syscall API，http 库又是构建在 net 库之上，所以在 Go 我们可以借助标准库，很轻松地写出高性能的 http 服务，下面是一个简单的 `hello world` 服务的代码：

```go
package main

import (
	"io"
	"log"
	"net/http"
)

func sayhello(wr http.ResponseWriter, r *http.Request) {
	wr.WriteHeader(200)
	io.WriteString(wr, "hello world")
}

func main() {
	http.HandleFunc("/", sayhello)
	err := http.ListenAndServe(":9090", nil)
	if err != nil {
		log.Fatal("ListenAndServe:", err)
	}
}
```

我们需要衡量一下这个 web 服务的吞吐量，再具体一些，实际上就是接口的 QPS。借助 wrk，在家用电脑 Macbook Pro 上对这个 `hello world` 服务进行基准测试，Mac 的硬件情况如下：

```shell
CPU: Intel(R) Core(TM) i5-5257U CPU @ 2.70GHz
Core: 2
Threads: 4

Graphics/Displays:
      Chipset Model: Intel Iris Graphics 6100
          Resolution: 2560 x 1600 Retina
    Memory Slots:
          Size: 4 GB
          Speed: 1867 MHz
          Size: 4 GB
          Speed: 1867 MHz
Storage:
          Size: 250.14 GB (250,140,319,744 bytes)
          Media Name: APPLE SSD SM0256G Media
          Size: 250.14 GB (250,140,319,744 bytes)
          Medium Type: SSD
```

测试结果：

```shell
~ ❯❯❯ wrk -c 10 -d 10s -t10 http://localhost:9090
Running 10s test @ http://localhost:9090
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   339.99us    1.28ms  44.43ms   98.29%
    Req/Sec     4.49k   656.81     7.47k    73.36%
  449588 requests in 10.10s, 54.88MB read
Requests/sec:  44513.22
Transfer/sec:      5.43MB

~ ❯❯❯ wrk -c 10 -d 10s -t10 http://localhost:9090
Running 10s test @ http://localhost:9090
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   334.76us    1.21ms  45.47ms   98.27%
    Req/Sec     4.42k   633.62     6.90k    71.16%
  443582 requests in 10.10s, 54.15MB read
Requests/sec:  43911.68
Transfer/sec:      5.36MB

~ ❯❯❯ wrk -c 10 -d 10s -t10 http://localhost:9090
Running 10s test @ http://localhost:9090
  10 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   379.26us    1.34ms  44.28ms   97.62%
    Req/Sec     4.55k   591.64     8.20k    76.37%
  455710 requests in 10.10s, 55.63MB read
Requests/sec:  45118.57
Transfer/sec:      5.51MB
```

多次测试的结果在 4w 左右的 QPS浮动，响应时间最多也就是 40ms 左右，对于一个 web 程序来说，这已经是很不错的成绩了，我们只是照抄了别人的示例代码，就完成了一个高性能的 `hello world` 服务器，是不是很有成就感？

这还只是家用 PC，线上服务器大多都是 24 核心起，32G 内存+，CPU 基本都是 Intel i7。所以同样的程序在服务器上运行会得到更好的结果。

这里的 `hello world` 服务没有任何业务逻辑。真实环境的程序要复杂得多，有些程序偏 Network-bound，例如一些 cdn 服务、proxy 服务；有些程序偏 CPU/GPU bound，例如登陆校验服务、图像处理服务；有些程序偏 Disk IO-bound，例如专门的存储系统，数据库。不同的程序瓶颈会体现在不同的地方，这里提到的这些功能单一的服务相对来说还算容易分析。如果碰到业务逻辑复杂代码量巨大的模块，其瓶颈并不是三下五除二可以推测出来的，还是需要从压力测试中得到更为精确的结论。

对于 IO/Network bound 类的程序，其表现是网卡/磁盘 IO 会先于 CPU 打满，这种情况即使优化 CPU 的使用也不能提高整个系统的吞吐量，只能提高磁盘的读写速度，增加内存大小，提升网卡的带宽来提升整体性能。而 CPU bound 类的程序，则是在存储和网卡未打满之前 CPU 占用率提前到达 100%，CPU 忙于各种计算任务，IO 设备相对则较闲。

无论哪种类型的服务，在资源使用到极限的时候都会导致请求堆积，超时，系统 hang 死，最终伤害到终端用户。对于分布式的 web 服务来说，瓶颈还不一定总在系统内部，也有可能在外部。非计算密集型的系统往往会在关系型数据库环节失守，而这时候 web 模块本身还远远未达到瓶颈。

不管我们的服务瓶颈在哪里，最终要做的事情都是一样的，那就是流量限制。

## 常见的流量限制手段

流量限制的手段有很多，最常见的：漏桶、令牌桶两种：

1. 漏桶是指我们有一个一直装满了水的桶，每过固定的一段时间即向外漏一滴水。如果你接到了这滴水，那么你就可以继续服务请求，如果没有接到，那么就需要等待下一滴水。
2. 令牌桶则是指匀速向桶中添加令牌，服务请求时需要从桶中获取令牌，令牌的数目可以按照需要消耗的资源进行对应的调整。如果没有令牌，可以选择等待，或者放弃。

这两种方法看起来很像，不过还是有区别的。漏桶流出的速率固定，而令牌桶只要在桶中有令牌，那就可以拿。也就是说令牌桶是允许一定程度的并发的，比如同一个时刻，有 100 个用户请求，只要令牌桶中有 100 个令牌，那么这 100 个请求全都会放过去。令牌桶在桶中没有令牌的情况下也会退化为漏桶模型。

实际应用中令牌桶应用较为广泛，开源界流行的限流器大多数都是基于令牌桶思想的。并且在此基础上进行了一定程度的扩充，比如 `github.com/juju/ratelimit` 提供了几种不同特色的令牌桶填充方式：

```go
func NewBucket(fillInterval time.Duration, capacity int64) *Bucket
```

默认的令牌桶，fillInterval 指每过多长时间向桶里放一个令牌，capacity 是桶的容量，超过桶容量的部分会被直接丢弃。桶初始是满的。

```go
func NewBucketWithQuantum(fillInterval time.Duration, capacity, quantum int64) *Bucket
```

和普通的 NewBucket 的区别是，每次向桶中放令牌时，是放 quantum 个令牌，而不是一个令牌。

```go
func NewBucketWithRate(rate float64, capacity int64) *Bucket
```

这个就有点特殊了，会按照提供的比例，每秒钟填充令牌数。例如 capacity 是 100，而 rate 是 0.1，那么每秒会填充 10 个令牌。

从桶中获取令牌也提供了几个 API：

```go
func (tb *Bucket) Take(count int64) time.Duration {}
func (tb *Bucket) TakeAvailable(count int64) int64 {}
func (tb *Bucket) TakeMaxDuration(count int64, maxWait time.Duration) (time.Duration, bool) {}
func (tb *Bucket) Wait(count int64) {}
func (tb *Bucket) WaitMaxDuration(count int64, maxWait time.Duration) bool {}
```

名称和功能都比较直观，这里就不再赘述了。相比于开源界更为有名的 google 的 Java 工具库 Guava 中提供的 ratelimiter，这个库不支持令牌桶预热，且无法修改初始的令牌容量，所以可能个别情况下的需求可能会无法满足。但在明白令牌桶的基本原理之后，如果没办法满足需求，相信你也可以很快对其进行修改并支持自己的业务场景。

## 原理

## 题外话: 分布式流量限制

TODO，在线系统应该保证 CPU 资源的余量，以确保 QoS，不应完全把资源用暴。