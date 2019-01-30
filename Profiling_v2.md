性能分析
===

StackExchange.Redis公开了一些方法和类型来启用性能分析。 由于其异步和多路复用
表现分析是一个有点复杂的主题。

接口
---

分析接口由 `ProfilingSession`, `ConnectionMultiplexer.RegisterProfiler(Func<ProfilingSession>)`,
`ProfilingSession.FinishProfiling()`, 和 `IProfiledCommand` 组合而成。

你可以用一个 `ConnectionMultiplexer` 实例注册一个会提供一个相关的 `ProfilingSession` 的回调 (`Func<ProfilingSession>`)。当需要时，类库会执行这个回调，
并且 `如果` 返回的不是一个空的 session: 操作会附加到这个 session 上， 调用某个特定的 session 的 `FinishProfiling` 会返回一个包含了所有发送给 `ConnectionMultiplexer` 配置的 redis 服务器要执行的命令相关时间信息的 `IProfiledCommand` 集合。
主要是由 callback 来维护追踪的独立的 session 需要的状态。

可用时间
---

StackExchange.Redis显示有关以下内容的信息：

- 涉及到的redis服务器
- 正在查询的redis数据库
- redis 运行命令
- 用于路由命令的标志
- 命令的初始创建时间
- 使命令进入队列所需的时间
- 命令入队后，发送命令需要多长时间
- 发送命令后，从redis接收响应需要多长时间
- 收到响应后处理响应所需的时间
- 如果命令是响应集群 ASK 或 MOVED 响应而发送的
  - 如果是，原始命令是什么

`TimeSpan` 有较高的精度，如果运行时支持。 `DateTime` 准确度如 `DateTime.UtcNow`。

性能分析示例
---

由于 StackExchange.Redis 的异步接口，分析需要外部协助将相关的命令组合在一起。
这是通过借助一个回调返回需要的 `ProfilingSession` 对象，之后调用这个 session 的 `FinishProfiling()` 来实现的。

可能最有用的通用会话提供程序是一个自动提供会话并在 `async` 调用之间工作的会话提供程序。 这很简单：

```c#
class AsyncLocalProfiler
{
    private readonly AsyncLocal<ProfilingSession> perThreadSession = new AsyncLocal<ProfilingSession>();

    public ProfilingSession GetSession()
    {
        var val = perThreadSession.Value;
        if (val == null)
        {
            perThreadSession.Value = val = new ProfilingSession();
        }
        return val;
    }
}
// ...
var profiler = new AsyncLocalProfiler();
multiplexer.RegisterProfiler(profiler.GetSession);
```

这将自动为每个异步上下文创建一个分析会话（如果有，则重新使用现有会话）。 在一些工作单元结束时，
调用代码可以使用
`var commands = profiler.GetSession（）。FinishProfiling（）;`
来获取执行的操作和时间信息数据。

---

一个将许多不同线程发出的命令关联在一起的 toy 示例（同时仍然允许不相关的工作不被记录）

``` csharp
class ToyProfiler
{
    // note this won't work over "await" boundaries; "AsyncLocal" would be necessary there
    private readonly ThreadLocal<ProfilingSession> perThreadSession = new ThreadLocal<ProfilingSession>();
    public ProfilingSession PerThreadSession
    {
        get => perThreadSession.Value;
        set => perThreadSession.Value = value;
    }
}

// ...

ConnectionMultiplexer conn = /* initialization */;
var profiler = new ToyProfiler();
var sharedSession = new ProfilingSession();

conn.RegisterProfiler(() => profiler.PerThreadSession);

var threads = new List<Thread>();

for (var i = 0; i < 16; i++)
{
    var db = conn.GetDatabase(i);

    var thread =
        new Thread(
            delegate()
            {
                // set each thread to share a session
                profiler.PerThreadSession = sharedSession;

                var threadTasks = new List<Task>();

                for (var j = 0; j < 1000; j++)
                {
                    var task = db.StringSetAsync("" + j, "" + j);
                    threadTasks.Add(task);
                }

                Task.WaitAll(threadTasks.ToArray());
            }
        );

    threads.Add(thread);
}

threads.ForEach(thread => thread.Start());
threads.ForEach(thread => thread.Join());

var timings = sharedSession.FinishProfiling();
```

最后，`timings` 将包含16,000个 `IProfiledCommand` 对象 - 每个发送给redis的命令对应一个对象。

如果你像下面这样做：

``` csharp
ConnectionMultiplexer conn = /* initialization */;
var profiler = new ToyProfiler();

conn.RegisterProfiler(() => profiler.PerThreadSession);

var threads = new List<Thread>();

var perThreadTimings = new ConcurrentDictionary<Thread, List<IProfiledCommand>>();

for (var i = 0; i < 16; i++)
{
    var db = conn.GetDatabase(i);

    var thread =
        new Thread(
            delegate()
            {
                var threadTasks = new List<Task>();
                profiler.PerThreadSession = new ProfilingSession();

                for (var j = 0; j < 1000; j++)
                {
                    var task = db.StringSetAsync("" + j, "" + j);
                    threadTasks.Add(task);
                }

                Task.WaitAll(threadTasks.ToArray());

                perThreadTimings[Thread.CurrentThread] = profiler.PerThreadSession.FinishProfiling().ToList();
            }
        );
    threads.Add(thread);
}

threads.ForEach(thread => thread.Start());
threads.ForEach(thread => thread.Join());
```

`perThreadTimings` 最终会有1000个 `IProfilingCommand` 的16个，主要是由 `Thread` 发出。

不再看这个示例，这里是如何在一个MVC5应用程序中配置 StackExchange.Redis。

首先针对你的 `ConnectionMultiplexer` 对象注册以下 `IProfiler`：

``` csharp
public class RedisProfiler
{
    const string RequestContextKey = "RequestProfilingContext";

    public ProfilingSession GetSession()
    {
        var ctx = HttpContext.Current;
        if (ctx == null) return null;

        return (ProfilingSession)ctx.Items[RequestContextKey];
    }

    public void CreateSessionForCurrentRequest()
    {
        var ctx = HttpContext.Current;
        if (ctx != null)
        {
            ctx.Items[RequestContextKey] = new ProfilingSession();
        }
    }
}
```

然后，将以下内容添加到Global.asax.cs文件中（其中`_redisProfiler`是profiler 的 *实例* ）：

``` csharp
protected void Application_BeginRequest()
{
    _redisProfiler.CreateSessionForCurrentRequest();
}

protected void Application_EndRequest()
{
    var session = _redisProfiler.GetSession();
    if (session != null)
    {
        var timings = session.FinishProfiling();

        // do what you will with `timings` here
    }
}
```

并且要确保连接创建的时候注册 profiler

```C#
connection.RegisterProfiler(() => _redisProfiler.GetSession());
```

这个实现将所有redis命令（包括 `async / await` -ed 命令）与触发它们的http请求分组。

[查看原文](https://github.com/StackExchange/StackExchange.Redis/blob/master/docs/Profiling_v2.md)
---