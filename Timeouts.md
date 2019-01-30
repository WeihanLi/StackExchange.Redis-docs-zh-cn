你是否正遇到网络或 CPU 的瓶颈？
==============

验证客户端和托管redis-server的服务器上支持的最大带宽。如果有请求被带宽限制，则它们需要更长时间才能完成，从而可能导致超时。
同样，验证您没有在客户端或服务器框上获得CPU限制，这将导致请求等待CPU时间，从而超时。

有没有命令需要在 redis 服务器上处理很长时间？
---------------

可能有一些命令需要很长时间才能在redis服务器上处理，导致请求超时。
长时间运行的命令的很少例子有 mget有大量的键，键*或写得不好的lua脚本。
可以运行通过 SlowLog 命令查看是否有请求花费比预期更长的时间。

在 [这里](http://redis.io/commands/slowlog) 可以找到关于命令的更多细节。

在向Redis发出的几个小请求之前是否有大的请求超时？
---------------

错误消息中的参数“qs”告诉您有多少从客户端发送到服务器，但尚未处理响应的请求。
对于某些类型的加载，您可能会看到此值不断增长，因为 StackExchange.Redis 使用单个TCP连接，并且一次只能读取一个响应。
即使第一个操作超时，它也不会停止 向服务器发送/从服务器发送 数据，其他请求也会被阻塞，直到该操作完成。 从而，导致超时。
一个解决方案是通过确保redis-server缓存对于您的工作负载足够大并将大值分割为更小的块来最小化超时的可能性。

另一个可能的解决方案是在客户端中使用 ConnectionMultiplexer 对象池，并在发送新请求时选择“最小化加载”ConnectionMultiplexer。 这样可能会防止单个超时导致其他请求也超时。

在超时异常中，是否有很多 busyio 或 busyworker 线程？
---------------

让我们先了解一下 ThreadPool 增长的一些细节：

CLR ThreadPool有两种类型的线程 - “工作线程”和“I/O 完成端口”（也称为 IOCP）线程。

- 工作线程用于处理 `Task.Run(...)` 或 `ThreadPool.QueueUserWorkItem(...)` 方法时。当工作需要在后台线程上发生时，这些线程也被CLR中的各种组件使用。
- 当异步IO发生时（例如从网络读取），使用IOCP线程。

线程池根据需要提供新的工作线程或I / O完成线程（无任何调节），直到达到每种类型线程的“最小”设置。 默认情况下，最小线程数设置为系统上的处理器数。

一旦现有（繁忙）线程的数量达到“最小”线程数，ThreadPool将调节每500毫秒向一个线程注入新线程的速率。 这意味着如果你的系统需要一个IOCP线程的工作，它会很快处理这个工作。 但是，如果工作突发超过配置的“最小”设置，那么在处理一些工作时会有一些延迟，因为ThreadPool会等待两个事情之一发生：

  1. 现有线程可以自由处理工作
  2. 连续 500ms 没有现有线程空闲，因此创建一个新线程。

基本上，这意味着当忙线程数大于最小线程时，在应用程序处理网络流量之前，可能需要付出500毫秒的延迟。 
此外，重要的是要注意，当现有线程保持空闲超过15秒（基于我记得），它将被清理，这个增长和收缩的循环可以重复。

如果我们看一个来自 StackExchange.Redis（build 1.0.450或更高版本）的示例错误消息，您将看到它现在会打印 ThreadPool 统计信息（请参阅下面的IOCP和WORKER详细信息）。

  ```
  System.TimeoutException: Timeout performing GET MyKey, inst: 2, mgr: Inactive, queue: 6, qu: 0, qs: 6, qc: 0, wr: 0, wq: 0, in: 0, ar: 0,
  IOCP: (Busy=6,Free=994,Min=4,Max=1000), 
  WORKER: (Busy=3,Free=997,Min=4,Max=1000)
  ```

在上面的示例中，您可以看到，对于 IOCP 线程，有6个忙线程，并且系统配置为允许4个最小线程。 在这种情况下，客户端可能会看到两个500毫秒的延迟，因为6> 4。

请注意，如果 IOCP 或 WORKER 线程的增长受到限制，StackExchange.Redis 可能会超时。

同样需要注意的是 如果你使用的 .NET Core 版本使用的 `netstandard` 版本小于 2.0，IOCP 和 WORKER 线程将不会显示。

建议：

鉴于上述信息，建议将 IOCP 和 WORKER 线程的最小配置值设置为大于默认值的值。 我们不能给出一个大小适合所有指导这个值应该是什么，因为对于一个应用程序的正确的值对于另一个应用程序而言总会太高/低。
此设置也会影响复杂应用程序的其他部分的性能，因此您需要根据您的特定需求调整此设置。 
一个好的起点是200或300，然后根据需要进行测试和调整。

如何配置这个设置：

- 在 ASP.NET 中，使用 machine.config 中 `<processModel>` 配置元素下的
[“minIoThreads”配置设置](https://msdn.microsoft.com/en-us/library/7w2sway1(v=vs.71).aspx)。
根据微软的做法，你不能修改每个站点 web.config 中的这个值（即使你过去这样做是可以的），如果你这样改的话你所有的.NET 站点都会使用这个设置的值。
请注意如果你设置 `autoconfig` 为 `false` 是不需要添加每一个属性的，仅需要添加 `autoconfig="false"` 并且覆盖原来的值就可以了：
`<processModel autoConfig="false" minIoThreads="250" />`

  > **重要说明：** 此配置元素中指定的值是为*每个核* 设置。例如，如果你有一个4核的机器，并希望你的 minIthreads 设置为200在运行时，你应该使用 `<processModel minIoThreads ="50"/>`。

- 在 ASP.NET 之外，使用 [ThreadPool.SetMinThreads(...)](https://msdn.microsoft.com//en-us/library/system.threading.threadpool.setminthreads(v=vs.100).aspx)API。

- 在 .NET Core 中 添加环境变量 `COMPlus_ThreadPool_ForceMinWorkerThreads` 来覆盖默认的 `MinThreads` 设置，参考 [Environment/Registry Configuration Knobs](https://github.com/dotnet/coreclr/blob/master/Documentation/project-docs/clr-configuration-knobs.md)

[查看原文](https://github.com/StackExchange/StackExchange.Redis/blob/master/docs/Timeouts.md)
---

[返回主页](./README.md)
---