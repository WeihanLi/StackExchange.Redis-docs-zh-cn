分析
===

StackExchange.Redis公开了一些方法和类型来启用性能分析。 由于其异步和多路复用
表现分析是一个有点复杂的主题。

接口
---

分析接口由 `IProfiler`, `ConnectionMultiplexer.RegisterProfiler(IProfiler)` ，`ConnectionMultiplexer.BeginProfiling(object)` ，
`ConnectionMultiplexer.FinishProfiling(object)` 和 `IProfiledCommand`。

你可以用一个 `ConnectionMultiplexer` 实例注册一个  `IProfiler` ，它不能被改变。你可以通过调用 `BeginProfiling(object)` 开始分析一个给定的上下文对象（例如，线程，Http请求等等），调用 `FinishProfiling(object)` 来完成。
`FinishProfiling(object)` 返回一个 `IProfiledCommand` 集合的对象，它包含了通过 `(Begin|Finish)Profiling` 调用和给定上下文配置好的 `ConnectionMultiplexer` 对象发送到 redis 的所有命令的时间信息。
 
应该使用什么“上下文”对象是应用程序来确定的。

可用时间
---

StackExchange.Redis显示有关以下内容的信息：

- 涉及到的redis服务器
- 正在查询的redis数据库
- redis命令运行
- 用于路由命令的标志
- 命令的初始创建时间
- 使命令进入队列所需的时间
- 命令入队后，发送命令需要多长时间
- 发送命令后，从redis接收响应需要多长时间
- 收到响应后处理响应所需的时间
- 如果命令是响应集群 ASK 或 MOVED 响应而发送的
    - 如果是，原始命令是什么

`TimeSpan` 有较高的精度，如果运行时支持。 `DateTime` 准确度如 `DateTime.UtcNow`。

选择上下文
---

由于StackExchange.Redis的异步接口，分析需要外部协助将相关的命令组合在一起。 这是实现的
通过提供上下文对象，当你开始和结束profiling（通过 `BeginProfiling(object)` ＆ `FinishProfiling(object)` 方法），当一个命令被发送（通过 `IProfiler` 接口的 `GetContext()` 方法）。

一个将许多不同线程发出的命令关联在一起的玩具示例：

```C#
class ToyProfiler : IProfiler
{
	public ConcurrentDictionary<Thread, object> Contexts = new ConcurrentDictionary<Thread, object>();

	public object GetContext()
	{
		object ctx;
		if(!Contexts.TryGetValue(Thread.CurrentThread, out ctx)) ctx = null;

		return ctx;
	}
}

// ...

ConnectionMultiplexer conn = /* initialization */;
var profiler = new ToyProfiler();
var thisGroupContext = new object();

conn.RegisterProfiler(profiler);

var threads = new List<Thread>();

for (var i = 0; i < 16; i++)
{
    var db = conn.GetDatabase(i);

    var thread =
        new Thread(
            delegate()
            {
                var threadTasks = new List<Task>();

                for (var j = 0; j < 1000; j++)
                {
                    var task = db.StringSetAsync("" + j, "" + j);
                    threadTasks.Add(task);
                }

                Task.WaitAll(threadTasks.ToArray());
            }
        );

	profiler.Contexts[thread] = thisGroupContext;

	threads.Add(thread);
}

conn.BeginProfiling(thisGroupContext);

threads.ForEach(thread => thread.Start());
threads.ForEach(thread => thread.Join());

IEnumerable<IProfiledCommand> timings = conn.FinishProfiling(thisGroupContext);
```

最后，`timings` 将包含16,000个 `IProfiledCommand` 对象 - 每个发送给redis的命令对应一个对象。

如果相反，你像下面这样做：

```C#
ConnectionMultiplexer conn = /* initialization */;
var profiler = new ToyProfiler();

conn.RegisterProfiler(profiler);

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

                conn.BeginProfiling(Thread.CurrentThread);

                for (var j = 0; j < 1000; j++)
                {
                    var task = db.StringSetAsync("" + j, "" + j);
                    threadTasks.Add(task);
                }

                Task.WaitAll(threadTasks.ToArray());

                perThreadTimings[Thread.CurrentThread] = conn.FinishProfiling(Thread.CurrentThread).ToList();
            }
        );

    profiler.Contexts[thread] = thread;

    threads.Add(thread);
}
                
threads.ForEach(thread => thread.Start());
threads.ForEach(thread => thread.Join());
```

`perThreadTimings` 最终会有1000个 `IProfilingCommand` 的16个，键由 `Thread` 发出。

不再看玩具示例，这里是如何在一个MVC5应用程序中配置 StackExchange.Redis。

首先针对你的 `ConnectionMultiplexer` 对象注册以下 `IProfiler`：

```C#
public class RedisProfiler : IProfiler
{
    const string RequestContextKey = "RequestProfilingContext";

    public object GetContext()
    {
        var ctx = HttpContext.Current;
        if (ctx == null) return null;

        return ctx.Items[RequestContextKey];
    }

    public object CreateContextForCurrentRequest()
    {
        var ctx = HttpContext.Current;
        if (ctx == null) return null;

        object ret;
        ctx.Items[RequestContextKey] = ret = new object();

        return ret;
    }
}
```

然后，将以下内容添加到 Global.asax.cs 文件：

```C#
protected void Application_BeginRequest()
{
    var ctxObj = RedisProfiler.CreateContextForCurrentRequest();
    if (ctxObj != null)
    {
        RedisConnection.BeginProfiling(ctxObj);
    }
}

protected void Application_EndRequest()
{
    var ctxObj = RedisProfiler.GetContext();
    if (ctxObj != null)
    {
        var timings = RedisConnection.FinishProfiling(ctxObj);
		
		// do what you will with `timings` here
    }
}
```

这个实现将所有redis命令（包括 `async / await` -ed命令）与触发它们的http请求分组。

[查看原文](https://github.com/StackExchange/StackExchange.Redis/blob/master/Docs/Profiling.md)
---