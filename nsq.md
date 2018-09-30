# NSQ

## 介绍（NSQ是什么）

### 官网文档

【官网文档】https://nsq.io/overview/quick_start.html

【中文翻译】 http://wiki.jikexueyuan.com/project/nsq-guide/

【Github】https://github.com/nsqio/nsq ，https://github.com/nsqio/go-nsq

### 概念

**NSQ** 是实时的分布式消息处理平台，其设计的目的是用来大规模地处理每天数以十亿计级别的消息。

NSQ 具有**分布式**和**去中心化**拓扑结构，该结构具有无单点故障、故障容错、高可用性以及能够保证消息的可靠传递的特征。 参见 [features & guarantees](http://wiki.jikexueyuan.com/project/nsq-guide/features_guarantees.html).

NSQ 非常容易配置和部署，且具有最大的灵活性，支持众多消息协议。另外，官方还提供了拆箱即用 Go 和 Python 库。如果读者有兴趣构建自己的客户端的话，还可以参考官方提供的[协议规范](http://wiki.jikexueyuan.com/project/nsq-guide/tcp_protocol_spec.html)。

## 功能特性和保障

### 特性

- 支持无 SPOF 的分布式拓扑
- 水平扩展(没有中间件，无缝地添加更多的节点到集群)
- 低延迟消息传递
- 结合负载均衡和多播消息路由风格
- 擅长面向流媒体(高通量)和工作(低吞吐量)工作负载
- 主要是内存中(除了高水位线消息透明地保存在磁盘上)
- 运行时发现消费者找到生产者服务([nsqlookupd](https://github.com/bitly/nsq/tree/master/nsqlookupd/README.md))
- 传输层安全性 (TLS)
- 数据格式不可知
- 一些依赖项(容易部署)和健全的，有界，默认配置
- 任何语言都有简单 TCP 协议支持客户端库
- HTTP 接口统计、管理行为和生产者(**不需要客户端库发布**)
- 为实时检测集成了 [statsd](https://github.com/etsy/statsd/)
- 健壮的集群管理界面 ([nsqadmin](https://github.com/bitly/nsq/tree/master/nsqadmin/README.md))

### 保障

对于任何分布式系统来说，都是通过智能权衡来实现目标。通过这些透明的可靠性指标，我们希望能使得 NSQ 在部署到产品上的行为是可达预期的。

### 消息不可持久化（默认）

虽然系统支持消息持久化存储在磁盘中（通过 `--mem-queue-size` ），不过默认情况下消息都在**内存**中.

如果将 `--mem-queue-size` 设置为 0，所有的消息将会存储到磁盘。我们不用担心消息会丢失，nsq 内部机制保证在程序关闭时将队列中的数据持久化到硬盘，重启后就会恢复。

NSQ 没有内置的复制机制，却有各种各样的方法管理这种权衡，比如部署拓扑结构和技术，在容错的时候从属并持久化内容到磁盘。

### 消息最少会被投递一次

如上所述，这个假设成立于 `nsqd` 节点没有错误。

因为各种原因，消息可以被投递多次（客户端超时，连接失效，重新排队，等等）。由客户端负责操作。

### 接收到的消息是无序的

不要依赖于投递给消费者的消息的顺序。

和投递消息机制类似，它是由重新队列(requeues)，内存和磁盘存储的混合导致的，实际上，节点间不会共享任何信息。

它是相对的简单完成**疏松队列**，（例如，对于某个消费者来说，消息是有次序的，但是不能给你作为一个整体跨集群），通过使用时间窗来接收消息，并在处理前排序（虽然为了维持这个变量，必须抛弃时间窗外的消息）。

### 消费者最终找出所有话题的生产者

这个服务([nsqlookupd](https://github.com/bitly/nsq/tree/master/nsqlookupd/README.md)) 被设计成最终一致性。`nsqlookupd` 节点不会维持状态，也不会回答查询。

网络分区并不会影响可用性，分区的双方仍然能回答查询。部署性拓扑可以显著的减轻这类问题

## 内部构造

NSQ 由 3 个守护进程组成：

- **nsqd** 是接收、队列和传送消息到客户端的守护进程。
- **nsqlookupd** 是管理的拓扑信息，并提供了最终一致发现服务的守护进程。
- **nsqadmin** 是一个 Web UI 来实时监控集群（和执行各种管理任务）。

在 NSQ中的数据流建模为一个消息数据流和消费者的树。一个**话题（topic）**是一个独特的数据流。一个 **通道（channel）** 是订阅了某个 **话题** 的消费者的逻辑分组。

![topics/channels](http://wiki.jikexueyuan.com/project/nsq-guide/images/internal1.gif)

单个的 **nsqd** 可以有很多的话题，每个话题也可以有多个通道。一个通道会接收到一个话题中所有消息的副本，启用组播方式的传输，使消息同时在每个通道的所有订阅用户间分发，从而实现负载均衡。

这些原语组成一个强大的框架，用于表示各种[简单和复杂的拓扑结构](http://nsq.io/deployment/topology_patterns.html)。

有关 NSQ 的设计的更多信息请参见[设计文档](http://nsq.io/overview/design.html)。

### 话题和通道

话题（topic）和通道（channel），NSQ 的核心基础，最能说明如何把 Go 语言的特点无缝地转化为系统设计。

Go 语言中的通道（channel）（为消除歧义以下简称为“go-chan”）是表示队列一种自然的方式，因此一个 NSQ 话题（topic）/通道（channel），其核心，只是一个缓冲的 go-chan `Message`指针。缓冲区的大小等于 `--mem-queue-size`的配置参数。

在懂了读数据后，发布消息到一个**话题**（topic）的行为涉及到：

1. **消息**（message）结构的初始化（和消息体的内存分配）
2. 获取 `话题（topic）` 时的读-锁；
3. 检查是否有发布的能力的读-锁；
4. 发布一个有缓存的 go-chan

从一个话题中的通道获取消息不能依赖于经典的 go-chan 语义，因为多个 goroutines 在一个 go-chan 上接收消息将会分发消息，而最终要的结果是复制每个消息到每一个通道（goroutine）。

替代的是，每个话题维护着 3 个主要的 goroutines。第一个被称为 `router`，它负责用来从 incoming go-chan 读取最近发布的消息，并把消息保存到队列中（内存或硬盘）。

第二个称为 `messagePump` 消息泵，是负责复制和推送消息到如上所述的通道。

第三个是负责 `DiskQueue`磁盘队列 IO 和将在后面讨论。

通道是要稍微复杂一点 ，但是共享一个单独的输入和单个输出的目标 go-chan (抽象出来的事实是，在内部，消息可能会在内存或磁盘上）：

![queue goroutine](http://wiki.jikexueyuan.com/project/nsq-guide/images/internal2.png)

此外，每个通道的维护负责 2 个时间排序优先级队列，用来实现传输中（in-flight）消息超时（第 2 个随行 goroutines 用于监视它们）。

并行化的提高是通过每个数据结构管理一个通道，而不是依靠 Go 运行时的全局定时器调度。

**注意**：在内部，Go 运行时使用一个单一优先级队列和 goroutine 来管理定时器。这支持（但不局限于）的整个 `time`package。它通常避免了需要一个用户空间的时间顺序的优先级队列，但要意识到这是一个很重要的一个有着单一锁的数据结构，有可能影响`GOMAXPROCS > 1` 的表现。请参考 [runtime/time.go](http://golang.org/src/pkg/runtime/time.go?s=1684:1787#L83) 

### Backend / DiskQueue

NSQ 的设计目标之一就是要限定保持在内存中的消息数。它通过 `DiskQueue` 透明地将溢出的消息写入到磁盘上（对于一个话题或通道而言，`DiskQueue` 拥有的第三个主要的 goroutine）。

由于内存队列只是一个 go-chan，把消息放到内存中显得不重要，如果可能的话，则退回到磁盘：

```go
for msg := range c.incomingMsgChan {
    select {
    case c.memoryMsgChan <- msg:
    default:
        err := WriteMessageToBackend(&msgBuf, msg, c.backend)
        if err != nil {
            // ... handle errors ...
        }
    }
}
```

说到 Go `select` 语句的优势在于用在短短的几行代码实现这个功能：`default` 语句只在 `memoryMsgChan` 已满的情况下执行。

NSQ 还有临时通道的概念。临时的通道将丢弃溢出的消息（而不是写入到磁盘）并在没有客户端订阅时消失。这是一个完美的 Go’s Interface 例子。话题和通道有一个结构成员声明为一个 `Backend` interface，而不是一个具体的类型。正常的话题和通道使用 `DiskQueue`，而临时通道连接在 `DummyBackendQueue`中，它实现了一个 no-op 的`Backend`后台。

### 降低 GC 的压力

在任何垃圾回收环境中，你可能会关注到吞吐量量（做无用功），延迟（响应），并驻留集合大小（footprint）。

Go 的1.2版本，GC 采用，mark-and-sweep (parallel), non-generational, non-compacting, stop-the-world 和 mostly precise。这主要是因为剩余的工作未完成（它预定于Go 1.3 实现）。

Go 的 GC 一定会不断改进，但普遍的真理是：**你创建的垃圾越少，收集的时间越少**。

首先，重要的是要了解 GC 在真实的工作负载下是如何表现。为此，**nsqd** 以 [statsd](https://github.com/etsy/statsd/) 格式发布的 GC 统计（伴随着其他的内部指标）。**nsqadmin** 显示这些度量的图表，让您洞察 GC 的影响，频率和持续时间：

![single node view](http://wiki.jikexueyuan.com/project/nsq-guide/images/internal3.png)

为了切实减少垃圾，你需要知道它是如何生成的。再次 Go toolchain 提供了答案：

1. 使用 [`testing`](http://golang.org/pkg/testing/) package 和 `go test -benchmem` 来 benchmark 热点代码路径。它分析每个迭代分配的内存数量（和 benchmark 运行可以用 [`benchcmp`](http://golang.org/misc/benchcmp) 进行比较）。
2. 编译时使用 go build -gcflags -m，会输出[逃逸分析](http://en.wikipedia.org/wiki/Escape_analysis)的结果。

考虑到这一点，下面的优化证明对 **nsqd** 是有用的:

1. 避免 `[]byte` 到 `string` 的转换
2. buffers 或 object 的重新利用（并且某一天可能面临 [`sync.Pool`](https://groups.google.com/forum/#!topic/golang-dev/kJ_R6vYVYHU) 又名 [issue 4720](https://code.google.com/p/go/issues/detail?id=4720)）
3. 预先分配 slices(在 `make` 时指定容量)并且总是知道其中承载元素的数量和大小
4. 对各种配置项目使用一些明智的限制（例如消息大小）
5. 避免装箱（使用 `interface{}`）或一些不必要的包装类型（例如一个多值的”go-chan” 结构体）
6. 避免在热点代码路径使用 `defer` (它也消耗内存)

### TCP 协议

NSQ 的 TCP 协议 [protocol_spec](http://nsq.io/clients/tcp_protocol_spec.html) 是一个这些 GC 优化概念发挥了很大作用的的例子。

该协议用含有长度前缀的帧构造，使其可以直接高效的编码和解码：

```
[x][x][x][x][x][x][x][x][x][x][x][x]...
|  (int32) ||  (int32) || (binary)
|  4-byte  ||  4-byte  || N-byte
------------------------------------...
    size      frame ID     data
```

由于提前知道了帧部件的确切类型与大小，我们避免了 [`encoding/binary`](http://golang.org/pkg/encoding/binary/) 便利 [`Read()`](http://golang.org/pkg/encoding/binary/#Read) 和 [`Write()`](http://golang.org/pkg/encoding/binary/#Write) 包装（以及它们外部 interface 的查询与转换），而是直接调用相应的 [`binary.BigEndian`](http://golang.org/pkg/encoding/binary/#ByteOrder) 方法。

为了减少 socket 的 IO 系统调用，客户端 `net.Conn` 都用 [`bufio.Reader`](http://golang.org/pkg/bufio/#Reader) 和[`bufio.Writer`](http://golang.org/pkg/bufio/#Writer) 包装。`Reader` 暴露了 [`ReadSlice()`](http://golang.org/pkg/bufio/#Reader.ReadSlice) ，它会重复使用其内部缓冲区。这几乎消除了从 socket 读出数据的内存分配，大大降低 GC 的压力。这可能是因为与大多数命令关联的数据不会被忽视（在边缘情况下，这是不正确的，数据是显示复制的）。

在一个更低的水平，提供一个 `MessageID` 被声明为 `[16]byte`，以便能够把它作为一个 `map` key（slice 不能被用作 map key）。然而，由于从 socket 读取数据存储为 `[]byte`，而不是通过分配字符串键产生垃圾，并避免从 slice 的副本拷贝的数组形式的`MessageID`， `unsafe` package 是用来直接把 slice 转换成一个 `MessageID`：

```
id := *(*nsq.MessageID)(unsafe.Pointer(&msgID))
```

**注**： 这是一个 *hack*。它将不是必要的，如果编译器优 和 [Issue 3512](https://code.google.com/p/go/issues/detail?id=3512) 解决这个问题。另外值得一读通过[issue 5376](https://code.google.com/p/go/issues/detail?id=5376)，其中谈到的“const like” `byte` 类型 与 `string` 类型可以互换使用，而不需要分配和复制。

同样，Go 标准库只提供了一个数字转换成 `string` 的方法。为了避免 `string` 分配，nsqd 使用一个自定义的10进制转换方法在 []byte 直接操作。

这些看似微观优化，但却包含了 TCP 协议中一些最热门的代码路径。总体而言，每秒上万消息的速度，对分配和开销的数目显著影响：

```
benchmark                    old ns/op    new ns/op    delta
BenchmarkProtocolV2Data           3575         1963  -45.09%

benchmark                    old ns/op    new ns/op    delta
BenchmarkProtocolV2Sub256        57964        14568  -74.87%
BenchmarkProtocolV2Sub512        58212        16193  -72.18%
BenchmarkProtocolV2Sub1k         58549        19490  -66.71%
BenchmarkProtocolV2Sub2k         63430        27840  -56.11%

benchmark                   old allocs   new allocs    delta
BenchmarkProtocolV2Sub256           56           39  -30.36%
BenchmarkProtocolV2Sub512           56           39  -30.36%
BenchmarkProtocolV2Sub1k            56           39  -30.36%
BenchmarkProtocolV2Sub2k            58           42  -27.59%
```

### HTTP

NSQ 的 HTTP API 是建立在 Go 的 net/http 包之上。因为它只是 [`net/http`](http://golang.org/pkg/net/http/)，它可以利用没有特殊的客户端库的几乎所有现代编程环境。

它的简单性掩盖了它的能力，作为 Go 的 HTTP tool-chest 最有趣的方面之一是广泛的调试功能支持。该 [`net/http/pprof`](http://golang.org/pkg/net/http/pprof/) 包直接集成了原生的 HTTP 服务器，暴露获取 CPU，堆，goroutine 和操作系统线程性能的 endpoints。这些可以直接从 `go` tool 找到：

```
$ go tool pprof http://127.0.0.1:4151/debug/pprof/profile
```

这对调试和分析一个运行的进程非常有价值！

此外，`/stats` endpoint 返回的指标以任何 JSON 或良好格式的文本来呈现，很容易使管理员能够实时从命令行监控：

```
$ watch -n 0.5 'curl -s http://127.0.0.1:4151/stats | grep -v connected'
```

这产生的连续输出如下：

```
[page_views     ] depth: 0     be-depth: 0     msgs: 105525994 e2e%: 6.6s, 6.2s, 6.2s
    [page_view_counter        ] depth: 0     be-depth: 0     inflt: 432  def: 0    re-q: 34684 timeout: 34038 msgs: 105525994 e2e%: 5.1s, 5.1s, 4.6s
    [realtime_score           ] depth: 1828  be-depth: 0     inflt: 1368 def: 0    re-q: 25188 timeout: 11336 msgs: 105525994 e2e%: 9.0s, 9.0s, 7.8s
    [variants_writer          ] depth: 0     be-depth: 0     inflt: 592  def: 0    re-q: 37068 timeout: 37068 msgs: 105525994 e2e%: 8.2s, 8.2s, 8.2s

[poll_requests  ] depth: 0     be-depth: 0     msgs: 11485060 e2e%: 167.5ms, 167.5ms, 138.1ms
    [social_data_collector    ] depth: 0     be-depth: 0     inflt: 2    def: 3    re-q: 7568  timeout: 402   msgs: 11485060 e2e%: 186.6ms, 186.6ms, 138.1ms

[social_data    ] depth: 0     be-depth: 0     msgs: 60145188 e2e%: 199.0s, 199.0s, 199.0s
    [events_writer            ] depth: 0     be-depth: 0     inflt: 226  def: 0    re-q: 32584 timeout: 30542 msgs: 60145188 e2e%: 6.7s, 6.7s, 6.7s
    [social_delta_counter     ] depth: 17328 be-depth: 7327  inflt: 179  def: 1    re-q: 155843 timeout: 11514 msgs: 60145188 e2e%: 234.1s, 234.1s, 231.8s

[time_on_site_ticks] depth: 0     be-depth: 0     msgs: 35717814 e2e%: 0.0ns, 0.0ns, 0.0ns
    [tail821042#ephemeral     ] depth: 0     be-depth: 0     inflt: 0    def: 0    re-q: 0     timeout: 0     msgs: 33909699 e2e%: 0.0ns, 0.0ns, 0.0ns
```

最后，每个 Go release 版本带来可观的 HTTP 性能提升[autobench](https://github.com/davecheney/autobench)。与 Go 的最新版本重新编译时，它总是很高兴为您提供免费的性能提升！

### 依赖

对于其它生态系统，Go 依赖关系管理（或缺乏）的哲学需要一点时间去适应。

NSQ 从一个单一的巨大仓库衍化而来的，包含相关的 imports 和小到未分离的内部 packages，完全遵守构建和依赖管理的最佳实践。

有两大流派的思想：

1. **Vendoring**: 拷贝正确版本的依赖到你的应用程序的仓库，并修改您的 import 路径来引用本地副本。
2. **Virtual Env**: 列出你在构建时所需要的依赖版本，产生一种原生的 `GOPATH` 环境变量包含这些固定依赖。

**注:** 这确实只适用于二进制包，因为它没有任何意义的一个导入的包，使中间的决定，如一种依赖使用的版本。

NSQ 使用 **gpm** 提供如上述2种的支持。

它的工作原理是在 [`Godeps`](https://github.com/bitly/nsq/blob/master/Godeps) 文件记录你的依赖，方便日后构建 GOPATH 环境。为了编译，它在环境里包装并执行的标准 Go toolchain。该 Godeps 文件仅仅是 JSON 格式，可以进行手工编辑。

### 测试

Go 提供了编写测试和基准测试的内建支持，这使用 Go 很容易并发操作进行建模，这是微不足道的建立起来的一个完整的实例 nsqd 到您的测试环境中。

然而，最初实现有可能变成测试问题的一个方面：全局状态。最明显的 offender 是运行时使用该持有 nsqd 的引用实例的全局变量，例如包含配置元数据和到 parent nsqd 的引用。

某些测试会使用短形式的变量赋值，无意中在局部范围掩盖这个全局变量，即 `nsqd := NewNSQd(...)` 。这意味着，全局引用没有指向了当前正在运行的实例，破坏了测试实例。

要解决这个问题，一个包含配置元数据和到 parent nsqd 的引用上下文结构被传来传去。到全局状态的所有引用都替换为本地的语境，允许 children（话题（topic），通道（channel），协议处理程序等）来安全地访问这些数据，使之更可靠的测试。

### 健壮性

一个面对不断变化的网络条件或突发事件不健壮的系统，不会是一个在分布式生产环境中表现良好的系统。

NSQ 设计和的方式是使系统能够容忍故障而表现出一致的，可预测的和令人吃惊的方式来实现。

总体理念是快速失败，把错误当作是致命的，并提供了一种方式来调试发生的任何问题。

但是，为了应对，你需要能够检测异常情况。

### 心跳和超时

NSQ 的 TCP 协议是面向 push 的。在建立连接，握手，和订阅后，消费者被放置在一个为 0 的 `RDY` 状态。当消费者准备好接收消息，它更新的 `RDY` 状态到准备接收消息的数量。NSQ 客户端库不断在幕后管理，消息控制流的结果。

每隔一段时间，**nsqd** 将发送一个心跳线连接。客户端可以配置心跳之间的间隔，但 **nsqd** 会期待一个回应在它发送下一个心掉之前。

组合应用级别的心跳和 RDY 状态，避免头阻塞现象，也可能使心跳无用（即，如果消费者是在后面的处理消息流的接收缓冲区中，操作系统将被填满，堵心跳）

为了保证进度，所有的网络 IO 时间上限势必与配置的心跳间隔相关联。这意味着，你可以从字面上拔掉之间的网络连接 nsqd 和消费者，它会检测并正确处理错误。

当检测到一个致命错误，客户端连接被强制关闭。在传输中的消息会超时而重新排队等待传递到另一个消费者。最后，错误会被记录并累计到各种内部指标。

### 管理 Goroutines

非常容易启动 goroutine。不幸的是，不是很容易以协调他们的清理工作。避免死锁也极具挑战性。大多数情况下这可以归结为一个顺序的问题，在上游 goroutine 发送消息到 go-chan 之前，另一个 goroutine 从 go-chan 上接收消息。

为什么要关心这些？这很显然，孤立的 goroutine 是内存泄漏。内存泄露在长期运行的守护进程中是相当糟糕的，尤其当期望的是你的进程能够稳定运行，但其它都失败了。

更复杂的是，一个典型的 nsqd 进程中有许多参与消息传递 goroutines。在内部，消息的“所有权”频繁变化。为了能够完全关闭，统计全部进程内的消息是非常重要的。

虽然目前还没有任何灵丹妙药，下列技术使它变得更轻松管理。

#### WaitGroups

[`sync`](http://golang.org/pkg/sync/) 包提供了 sync.WaitGroup, 可以被用来累计多少个 goroutine 是活跃的（并且意味着一直等待直到它们退出）。

为了减少典型样板，nsqd 使用以下装饰器：

```
type WaitGroupWrapper struct {
    sync.WaitGroup
}

func (w *WaitGroupWrapper) Wrap(cb func()) {
    w.Add(1)
    go func() {
        cb()
        w.Done()
    }()
}

// can be used as follows:
wg := WaitGroupWrapper{}
wg.Wrap(func() { n.idPump() })
...
wg.Wait()
```

#### 退出信号

有一个简单的方式在多个 child goroutine 中触发一个事件是提供一个 go-chane，当你准备好时关闭它。所有在那个 go-chan 上挂起的 go-chan 都将会被激活，而不是向每个 goroutine 中发送一个单独的信号。

```
func work() {
    exitChan := make(chan int)
    go task1(exitChan)
    go task2(exitChan)
    time.Sleep(5 * time.Second)
    close(exitChan)
}
func task1(exitChan chan int) {
    <-exitChan
    log.Printf("task1 exiting")
}

func task2(exitChan chan int) {
    <-exitChan
    log.Printf("task2 exiting")
}
```

#### 退出时的同步

实现一个可靠的，无死锁的，所有传递中的消息的退出路径是相当困难的。一些提示：

1. 理想的情况是负责发送到 go-chan 的 goroutine 中也应负责关闭它。
2. 如果 message 不能丢失，确保相关的 go-chan 被清空（尤其是无缓冲的！），以保证发送者可以取得进展。
3. 另外，如果消息是不重要的，发送给一个单一的 go-chan 应转换为一个 `select` 附加一个退出信号（如上所述），以保证取得进展。
4. 一般的顺序应该是
   1. 停止接受新的连接（close listeners）
   2. 发送退出信号给 c hild goroutines (如上文)
   3. 在 `WaitGroup` 等待 goroutine 退出（如上文）
   4. 恢复缓冲数据
   5. 刷新所有东西到硬盘

#### 日志

最后，日志是您所获得的记录 goroutine 进入和退出的重要工具！。这使得它相当容易识别造成死锁或泄漏的情况的罪魁祸首。

**nsqd** 日志行包括 goroutine 与他们的 siblings(and parent）的信息，如客户端的远程地址或话题（topic）/通道（channel）名。

该日志是详细的，但不是详细的日志是压倒性的。有一条细线，但 nsqd 倾向于发生故障时在日志中提供更多的信息，而不是试图减少繁琐的有效性为代价。



## 内部组件介绍

### **nsqlookupd**

是负责管理拓扑信息，并提供了最终一致发现服务的守护进程。客户端通过查询 nsqlookupd 来发现指定话题（topic）的生产者，并且 nsqd 节点广播话题（topic）和通道（channel）信息 。

简单的说nsqlookupd就是中心管理服务，它使用tcp(默认端口4160)管理nsqd服务，使用http(默认端口4161)管理nsqadmin服务。同时为客户端提供查询功能。

总的来说，**nsqlookupd具有以下功能或特性**

- 唯一性，在一个Nsq服务中只有一个nsqlookupd服务。当然也可以在集群中部署多个nsqlookupd，但它们之间是没有关联的
- 去中心化，即使nsqlookupd崩溃，也会不影响正在运行的nsqd服务
- 充当nsqd和naqadmin信息交互的中间件
- 提供一个http查询服务，给客户端定时更新nsqd的地址目录 

### nsqadmin

是一个 Web UI 来实时监控集群（和执行各种管理任务）。

总的来说，**nsqadmin具有以下功能或特性**

- 提供一个对topic和channel统一管理的操作界面以及各种实时监控数据的展示，界面设计的很简洁，操作也很简单
- 展示所有message的数量，恩....装X利器
- 能够在后台创建topic和channel，这个应该不常用到
- nsqadmin的所有功能都必须依赖于nsqlookupd，nsqadmin只是向nsqlookupd传递用户操作并展示来自nsqlookupd的数据

nsqadmin默认的访问地址是<http://127.0.0.1:4171/> 

### nsqd

**是一个负责接收、队列和传送消息到客户端的守护进程。**

简单的说，真正干活的就是这个服务，它主要负责message的收发，队列的维护。nsqd会默认监听一个tcp端口(4150)和一个http端口(4151)以及一个可选的https端口

总的来说，nsqd 具有以下功能或特性

- 对订阅了同一个topic，同一个channel的消费者使用负载均衡策略（不是轮询）
- 只要channel存在，即使没有该channel的消费者，也会将生产者的message缓存到队列中（注意消息的过期处理）
- 保证队列中的message至少会被消费一次，即使nsqd退出，也会将队列中的消息暂存磁盘上(结束进程等意外情况除外)
- 限定内存占用，能够配置nsqd中每个channel队列在内存中缓存的message数量，一旦超出，message将被缓存到磁盘中
- topic，channel一旦建立，将会一直存在，要及时在管理台或者用代码清除无效的topic和channel，避免资源的浪费

这是官方的图，第一个channel(meteics)因为有多个消费者，所以触发了负载均衡机制。后面两个channel由于没有消费者，所有的message均会被缓存在相应的队列里，直到消费者出现。



![nsqd clients](https://f.cloud.github.com/assets/187441/1700696/f1434dc8-6029-11e3-8a66-18ca4ea10aca.gif)

这里想到一个问题是，如果一个channel只有生产者不停的在投递message，会不会导致服务器资源被耗尽？也许nsqd内部做了相应处理，但还是要避免这种情况的出现 。

### 其他工具

#### nsq_stat

为所有的话题（topic）和通道（channel）的生产者轮询 `/stats`，并显示统计数据：

```
---------------depth---------------+--------------metadata---------------
  total    mem    disk inflt   def |     req     t-o         msgs clients
  24660  24660       0     0    20 |  102688       0    132492418       1
  25001  25001       0     0    20 |  102688       0    132493086       1
  21132  21132       0     0    21 |  102688       0    132493729       1
```

**命令行参数**

```shell
-channel string
    nsq通道
    NSQ channel
-count value
    计数显示
    number of reports/
-http-client-connect-timeout duration
	HTTP超时连接时间间隔（默认两秒）
    timeout for HTTP connect (default 2s)
-http-client-request-timeout duration
	HTTP请求超时时间间隔（默认两秒）
    timeout for HTTP request (default 5s)
-interval duration
	轮询和打印输出之间的时间间隔（默认两秒）
    duration of time between polling/printing output (default 2s)
-lookupd-http-address value
	lookupd的HTTP地址（可以有多个IP值）
    lookupd HTTP address (may be given multiple times)
-nsqd-http-address value
	nsqd的HTTP地址（可以有多个IP值）
    nsqd HTTP address (may be given multiple times)
-topic string
	NSQ 主题
    NSQ topic
-version
	打印NSQ版本
    print version
```

#### nsq_tail

消费指定的话题（topic）/通道（channel），并写到 stdout (和 tail(1) 类似)。

**命令行参数**

```shell
-channel string
 	NSQ通道
    NSQ channel
-consumer-opt value
	选择传递给 nsq.Consumer（可能会给多个值, 可参考http://godoc.org/github.com/bitly/go-nsq#Config)
    option to passthrough to nsq.Consumer (may be given multiple times, http://godoc.org/github.com/nsqio/go-nsq#Config)
-lookupd-http-address value
    lookupd HTTP 地址 (可以有多个值)
-max-in-flight int
	允许的最大消息数目（默认200）
    max number of messages to allow in flight (default 200)
-n int
	显示总的消息（如果空闲会继续等待）
    total messages to show (will wait if starved)
-nsqd-tcp-address value
	NSQ HTTP 地址 (可以有多个值)
    nsqd TCP address (may be given multiple times)
-topic string
	NSQ 主题
    NSQ topic
-version
	打印NSQ 版本
    print version string
```

#### nsq_to_file

消费指定的话题（topic）/通道（channel），并写到文件中，有选择的滚动和/或压缩文件。

**命令行参数**

```shell
-channel string
	nsq 通道（默认是"nsq_to_file"）
    nsq channel (default "nsq_to_file")
-consumer-opt value
	传递给 nsq.Consumer 的参数 (可能会给多次, http://godoc.org/github.com/bitly/go-nsq#Config)
    option to passthrough to nsq.Consumer (may be given multiple times, http://godoc.org/github.com/nsqio/go-nsq#Config)
-datetime-format string
	和 filename 里 <DATETIME> 格式兼容
    strftime compatible format for <DATETIME> in filename format (default "%Y-%m-%d_%H")
-filename-format string
	"<TOPIC>.<HOST><GZIPREV>.<DATETIME>.log": output 文件名格式 (<TOPIC>, <HOST>, <DATETIME>, <GZIPREV> 重新生成. <GZIPREV> 是当已经存在的 gzip 文件的前缀)
    output filename format (<TOPIC>, <HOST>, <PID>, <DATETIME>, <REV> are replaced. <REV> is increased when file already exists) (default "<TOPIC>.<HOST><REV>.<DATETIME>.log")
-gzip
	gzip 输出文件
    gzip output files.
-gzip-level int
	gzip 压缩级别 (1-9, 1=BestSpeed, 9=BestCompression)
    gzip compression level (1-9, 1=BestSpeed, 9=BestCompression) (default 6)
-host-identifier string
	输出到 log 文件，提供主机名。 <SHORT_HOST> 和 <HOSTNAME> 是有效的替换者
    value to output in log filename in place of hostname. <SHORT_HOST> and <HOSTNAME> are valid replacement tokens
-http-client-connect-timeout duration
    timeout for HTTP connect (default 2s)
-http-client-request-timeout duration
    timeout for HTTP request (default 5s)
-lookupd-http-address value
	lookupd HTTP 地址 (可能会给多次)
    lookupd HTTP address (may be given multiple times)
-max-in-flight int
	最大的消息数 to allow in flight
    max number of messages to allow in flight (default 200)
-nsqd-tcp-address value
	nsqd TCP 地址 (可能会给多次)
    nsqd TCP address (may be given multiple times)
-output-dir string
	输出文件所在的文件夹
    directory to write output files to (default "/tmp")
-rotate-interval duration
	每次时间间隔循环文件
    rotate the file every duration
-rotate-size rotate-size
	增长的循环文件大小
    rotate the file when it grows bigger than rotate-size bytes
-skip-empty-files
	跳过写空文件
    Skip writing empty files
-topic value
    nsq topic (may be given multiple times)
-topic-pattern string
	只输出一下模板的主题
    Only log topics matching the following pattern
-topic-refresh duration
	刷新主题的时间间隔
    how frequently the topic list should be refreshed (default 1m0s)
-version
	版本信息
    print version string
```

#### nsq_to_http

消费指定的话题（topic）/通道（channel）和执行 HTTP requests (GET/POST) 到指定的端点。

**命令行参数**

```shell
-channel string
	nsq 通道（channel）
    nsq channel (default "nsq_to_http")
-consumer-opt value
	参数，通过 nsq.Consumer (可能会给多次, http://godoc.org/github.com/bitly/go-nsq#Config)
    option to passthrough to nsq.Consumer (may be given multiple times, http://godoc.org/github.com/nsqio/go-nsq#Config)
-content-type string
	Content-Type 使用d for POST requests
    the Content-Type used for POST requests (default "application/octet-stream")
-get value
	HTTP 地址 to make a GET request to. '%s' will be printf replaced with data (可能会给多次)
    HTTP address to make a GET request to. '%s' will be printf replaced with data (may be given multiple times)
-http-client-connect-timeout duration
	HTTP连接超时时间（默认两秒）
    timeout for HTTP connect (default 2s)
-http-client-request-timeout duration
	HTTP请求超时时间（默认20秒）
    timeout for HTTP request (default 20s)
-lookupd-http-address value
	lookupd HTTP 地址 (可能会给多次)
    lookupd HTTP address (may be given multiple times)
-max-in-flight int
	最大的消息数 to allow in flight
    max number of messages to allow in flight (default 200)
-mode string
    the upstream request mode options: round-robin, hostpool (default), epsilon-greedy (default "hostpool")
-n int
	并发发布者的数量（默认100）
    number of concurrent publishers (default 100)
-nsqd-tcp-address value
	 nsqd TCP 地址 (可能会给多次)
    nsqd TCP address (may be given multiple times)
-post value
	HTTP 地址 to make a POST request to.  data will be in the body (可能会给多次)
    HTTP address to make a POST request to.  data will be in the body (may be given multiple times)
-sample float
    % of messages to publish (float b/w 0 -> 1) (default 1)
-status-every int
    the # of requests between logging status (per handler), 0 disables (default 250)
-topic string
	nsq 话题（topic）
    nsq topic
-version
	版本信息
    print version string

```

#### nsq_to_nsq

消费者指定的话题/通道和重发布消息到目的地 `nsqd` 通过 TCP。

**命令行参数**

```shell
-channel string
	nsq 通道（channel）
    nsq channel (default "nsq_to_nsq")
-consumer-opt value
	参数，通过 nsq.Consumer (可能会给多次, see http://godoc.org/github.com/bitly/go-nsq#Config)
    option to passthrough to nsq.Consumer (may be given multiple times, see http://godoc.org/github.com/nsqio/go-nsq#Config)
-destination-nsqd-tcp-address value
	nsqd TCP 目标地址 (可能会给多次)
    destination nsqd TCP address (may be given multiple times)
-destination-topic string
	nsq目标topic
    destination nsq topic
-lookupd-http-address value
	lookupd HTTP 地址 (可能会给多次)
    lookup d HTTP address (may be given multiple times)
-max-in-flight int
	允许 flight 最大的消息数（默认200）
    max number of messages to allow in flight (default 200)
-mode string
	上行请求的参数: round-robin (默认), hostpool
    the upstream request mode options: round-robin, hostpool (default), epsilon-greedy (default "hostpool")
-nsqd-tcp-address value
	nsqd TCP 地址 (可能会给多次)
    nsqd TCP address (may be given multiple times)
-producer-opt value
	传递到 nsq.Producer (可能会给多次, 参见 http://godoc.org/github.com/bitly/go-nsq#Config)
    option to passthrough to nsq.Producer (may be given multiple times, see http://godoc.org/github.com/nsqio/go-nsq#Config)
-require-json-field string
	JSON 消息: 仅传递消息，包含这个参数
    for JSON messages: only pass messages that contain this field
-require-json-value string
	JSON 消息: 仅传递消息要求参数有这个值 
    for JSON messages: only pass messages in which the required field has this value
-status-every int
	请求日志的状态(每个目的地), 0 不可用
    the # of requests between logging status (per destination), 0 disables (default 250)
-topic string
	nsq 话题（topic）
    nsq topic
-version
	打印版本信息
    print version string
-whitelist-json-field value
	JSON 消息: 传递这个字段 (可能会给多次)
    for JSON messages: pass this field (may be given multiple times)

```

#### to_nsq

采用 stdin 流，并分解到新行（默认），通过 TCP 重新发布到目的地 `nsqd`。

**命令行参数**

```shell
-delimiter string
	分割字符串(默认'\n')
    character to split input from stdin (default "\n")
-nsqd-tcp-address value
	目的地 nsqd TCP 地址 (可能会给多次)
    destination nsqd TCP address (may be given multiple times)
-producer-opt value
	参数，通过 nsq.Producer (可能会给多次, http://godoc.org/github.com/bitly/go-nsq#Config)
    option to passthrough to nsq.Producer (may be given multiple times, http://godoc.org/github.com/nsqio/go-nsq#Config)
-rate int
    Throttle messages to n/second. 0 to disable
-topic string
	发布到的 NSQ 话题（topic）
    NSQ topic to publish to
```

## 安装使用

提供window、mac、linux二进制安装包和源码安装。请参考以下官网链接。

https://nsq.io/deployment/installing.html

可选择对应平台和相应版本（此处试验使用的是1.0版）进行下载安装，把二进制可执行文件，并放在对应的环境变量目录下。也可以根据使用docker下载镜像进行编排使用。我是直接使用docker-compose进行启动服务搭建的测试环境。也可以使用二进制可执行文件开启服务进行测试。

### Docker-Compose安装部署

```yaml
# docker-compose.yml
version: '3'
services:
  nsqlookupd:
    image: nsqio/nsq
    command: /nsqlookupd
    ports:
      - 4160:4160
      - 4161:4161
  nsqd:
    image: nsqio/nsq
    command: /nsqd --lookupd-tcp-address=nsqlookupd:4160
    depends_on:
      - nsqlookupd
    ports:
      - 4150:4150
      - 4151:4151
  nsqadmin:
    image: nsqio/nsq
    command: /nsqadmin --lookupd-http-address=nsqlookupd:4161
    depends_on:
      - nsqlookupd
    ports:
      - 4171:4171
```

 运行docker-compose.yml文件下载image启动containers

```shell
docker-compose up -d
```

查看运行containers

```shell
docker-compose ps
```

查看containers的log

```shell
docker-compose logs
```

服务启动之后可以使用命令监测nsqd服务的运行状态

```shell
watch -n 0.5 "curl -s http://127.0.0.1:4151/stats"
```

如下图和文字示例：

以下是我监测到的三个topic的运行情况：

![1528871514646](C:\Users\ADMINI~1\AppData\Local\Temp\1528871514646.png)

```
nsqd v1.0.0-compat (built w/go1.8)
start_time 2018-06-05T09:56:59Z
uptime 23h55m26.48614264s

Health: OK

   [test           ] depth: 0     be-depth: 0     msgs: 177      e2e%: 
      [test-channel             ] depth: 0     be-depth: 0     inflt: 0    def: 0    re-q: 0     timeout: 0     msgs: 177      e2e%: 
        [V2 SKY-20180523GKH      ] state: 3 inflt: 0    rdy: 1    fin: 9        re-q: 0        msgs: 9        connected: 19m11s

   [write_test     ] depth: 1     be-depth: 0     msgs: 1        e2e%: 

   [write_test001  ] depth: 1     be-depth: 0     msgs: 1        e2e%: 
```

### 代码示例

$GOPATH/src/nsqdemo/**nsqproducer.go**

```go
// Nsq发送测试
package main

// NSQ生产者

import (
	"bufio"
	"fmt"
	"github.com/nsqio/go-nsq"
	"os"
)

var producer *nsq.Producer

// 主函数
func main() {
	IP1 := "localhost"
	strIP1 := IP1 + ":4150"

	InitProducer(strIP1)

	running := true

	//读取控制台输入
	reader := bufio.NewReader(os.Stdin)
	for running {
		data, _, _ := reader.ReadLine()
		command := string(data)
		if command == "stop" {
			running = false
		}
		// 向topic里推送消息
		err := Publish("test", command)
		if err != nil {
			panic(err)
		}

	}
	//关闭
	producer.Stop()
}

// 初始化生产者
func InitProducer(str string) {
	var err error
	fmt.Println("address: ", str)
	producer, err = nsq.NewProducer(str, nsq.NewConfig())
	if err != nil {
		panic(err)
	}
}

//	发布消息
func Publish(topic string, message string) error {
	var err error
	if producer != nil {
		if message == "" { //不能发布空串，否则会导致error
			return nil
		}
		err = producer.Publish(topic, []byte(message)) // 发布消息
		return err
	}
	return fmt.Errorf("producer is nil", err)
}
```

$GOPATH/src/nsqdemo/**nsqconsumer.go**

```go
//Nsq接收测试
package main

// NSQ消费者

import (
	"fmt"
	"time"

	"github.com/nsqio/go-nsq"
)

// 消费者
type ConsumerT struct{}

// 主函数
func main() {
	IP := "localhost"
	Addr := IP + ":4161"
    // 初始化消费者topic和channel
	InitConsumer("test", "test-channel", Addr)
	for {
		time.Sleep(time.Second * 10)
	}
}

// 处理消息
func (consumer *ConsumerT) HandleMessage(msg *nsq.Message) error {
	fmt.Println("receive", msg.NSQDAddress, "message:", string(msg.Body))
	return nil
}

// 初始化消费者
func InitConsumer(topic string, channel string, address string) {
	// 初始化nsq配置
	cfg := nsq.NewConfig()
	// NOTE: when not using nsqlookupd, LookupdPollInterval represents the duration of time between
	// reconnection attempts
	// 设置重连LookUpd时间 默认 1min
	cfg.LookupdPollInterval = time.Second

	c, err := nsq.NewConsumer(topic, channel, cfg) // 新建一个消费者
	if err != nil {
		panic(err)
	}
	fmt.Println("consumer = ", c, err)
	//屏蔽系统日志
	c.SetLogger(nil, 0)

	// 添加消费者接口
	c.AddHandler(&ConsumerT{})

	// c.AddHandler(nsq.HandlerFunc(HandleMessage))

	//建立NSQLookupd连接
	if err := c.ConnectToNSQLookupd(address); err != nil {
		panic(err)
	}
	// 建立一个nsqd连接
	if err = c.ConnectToNSQD(IP + ":4150"); err != nil {
		panic(err)
	}
}

```

### 操作使用

分别运行两个go文件：`go run nsqproducer.go` `go run nsqconsumer.go`

`producer` 这边输入字符串回车则会send一条message，`consumer`这边则会接收到message进行消费

![producer](https://raw.githubusercontent.com/yuyongbo/images/master/nsq/nsq_produce.png)

![consumer](https://raw.githubusercontent.com/yuyongbo/images/master/nsq/nsq_consumer.png)

![](https://raw.githubusercontent.com/yuyongbo/images/master/nsq/nsq_run.png)



## 终端使用

1. 在另外一个 shell 中，运行 `nsqlookupd`:

   ```
   $ nsqlookupd
   ```

2. 再开启一个 shell，运行 `nsqd`:

   ```
   $ nsqd --lookupd-tcp-address=127.0.0.1:4160
   ```

3. 再开启第三个 shell，运行 `nsqadmin`:

   ```
   $ nsqadmin --lookupd-http-address=127.0.0.1:4161
   ```

4. 开启第四个 shell，推送一条初始化数据(并且在集群中创建一个 topic):

   ```
   $ curl -d 'hello world 1' 'http://127.0.0.1:4151/put?topic=test'
   ```

5. 最后，开启第五个 shell， 运行 `nsq_to_file`:

   ```
   $ nsq_to_file --topic=test --output-dir=/tmp --lookupd-http-address=127.0.0.1:4161
   ```

6. 推送更多地数据到 `nsqd`:

   ```
   $ curl -d 'hello world 2' 'http://127.0.0.1:4151/put?topic=test'
   $ curl -d 'hello world 3' 'http://127.0.0.1:4151/put?topic=test'
   ```

7. 运行示意图如下所示

   ![image](https://raw.githubusercontent.com/yuyongbo/images/master/nsq/nsq_6.gif)


## 为什么要使用NSQ

[三大MQ的统计详情对比](https://stackshare.io/stackups/kafka-vs-nsq-vs-rabbitmq) (RabbitMQ vs. Kafka vs.NSQ)

https://segmentfault.com/a/1190000009194607

最近一直在寻找一个高性能，高可用的消息队列做内部服务之间的通讯。一开始想到用zeromq，但在查找资料的过程中，意外的发现了Nsq这个由golang开发的消息队列，毕竟是golang原汁原味的东西，功能齐全，关键是性能还不错。其中支持动态拓展，消除单点故障等特性，  都可以很好的满足我的需求

下面上一张Nsq与其他mq的对比图，看上去的确强大。下面简单记录一下Nsq的使用方法
![golang2017开发者大会](https://segmentfault.com/img/bVMKft?w=1238&h=492)
图片来自golang2017开发者大会

## 到底什么时候该使用MQ？

可参考58沈剑老师的博客 [到底什么时候该使用MQ](https://mp.weixin.qq.com/s/Brd-j3IcljcY7BV01r712Q##)
