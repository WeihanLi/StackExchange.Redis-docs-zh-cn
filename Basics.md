基本使用
===

StackExchange.Redis 中核心对象是在 `StackExchange.Redis` 命名空间中的 `ConnectionMultiplexer`  类，这个对象隐藏了多个服务器的详细信息。
因为`ConnectionMultiplexer`要做很多事，它被设计为在调用者之间可以**共享**和**重用**。
你不应该在执行每一个操作的时候就创建一个 `ConnectionMultiplexer`. 它完全是线程安全的，并准备好这种用法（多线程）。
在后续所有的例子中，我们假设你有一个`ConnectionMultiplexer`类的实例保存以重用。 
但现在，让我们来先创建一个。 这是使用 `ConnectionMultiplexer.Connect` 或 `ConnectionMultiplexer.ConnectAsync` 完成的，传递配置字符串或`ConfigurationOptions` 对象。
配置字符串可以采用逗号分隔的一系列节点的形式，所以让我们在默认端口（6379）上连接到本地机器上的一个实例：

``` C#
using StackExchange.Redis;
...
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost");
// ^^^ store and re-use this!!!
```

请注意，`ConnectionMultiplexer` 实现了 `IDisposable` 接口而且可以在不再需要的时候处理释放掉。
这是故意不展示使用 `using` 语句用法，因为你想要只是简单地使用一个`ConnectionMultiplexer`的情况是极少见的 ，因为想法是重用这个对象。

更复杂的情况可能涉及主/从设置; 对于此用法，只需简单的指定组成逻辑redis层的所有所需节点（它将自动标识主节点）

``` C#
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("server1:6379,server2:6379");
```

如果它发现两个节点都是主节点，则可以可选地指定可以用于解决问题的仲裁密钥，然而幸运地是这样的条件是非常罕见的。

一旦你有一个  `ConnectionMultiplexer` ，你可能有3个主要想做的事：

- 访问一个 redis 数据库（注意，在集群的情况下，单个逻辑数据库可以分布在多个节点上）
- 使用 redis 的的 [发布/订阅](http://redis.io/topics/pubsub) 功能
- 访问单独的服务器以进行维护/监视

使用 redis 数据库
---

访问redis数据库非常简单：

```C#
IDatabase db = redis.GetDatabase();
```

从 `GetDatabase` 返回的对象是一个成本很低的通道对象，不需要存储。
注意，redis支持多个数据库（虽然“集群”不支持），这可以可选地在调用 `GetDatabase` 中指定。
此外，如果您计划使用异步API，您需要 [`Task.AsyncState`][2] 有一个值，也可以指定：

```C#
int databaseNumber = ...
object asyncState = ...
IDatabase db = redis.GetDatabase(databaseNumber, asyncState);
```

一旦你有了`IDatabase`，它只是一个使用 [redis API](http://redis.io/commands) 的情况。
注意，所有方法都具有同步和异步实现。 
根据微软的命名指导，异步方法都以 `...Async(...)` 结尾，并且完全是可以等待的 `await` 等。

最简单的操作也许是存储和检索值：

```C#
string value = "abcdefg";
db.StringSet("mykey", value);
...
string value = db.StringGet("mykey");
Console.WriteLine(value); // writes: "abcdefg"
```

需要注意的是，这里的 `String...` 前缀表示 [redis 的 String 类型](http://redis.io/topics/data-types)，尽管它和 [.NET 的字符串类型][3] 都可以保存文本， 它们还是有较大差别的。
然而，redis 还允许键和值使用原始的二进制数据，用法和字符串是一样的：

```C#
byte[] key = ..., value = ...;
db.StringSet(key, value);
...
byte[] value = db.StringGet(key);
```

覆盖所有redis数据类型的所有 [redis数据库命令](http://redis.io/commands) 的都是可以使用的。

使用 redis 发布/订阅
----

redis的另一个常见用法是作为 [发布/订阅消息](http://redis.io/topics/pubsub)分发工具;
这也很简单，并且在连接失败的情况下， `ConnectionMultiplexer` 将处理重新订阅所请求的信道的所有细节。

```C#
ISubscriber sub = redis.GetSubscriber();
```

同样，从 `GetSubscriber` 返回的对象是一个不需要存储的低成本的通道对象。
发布/订阅 API没有数据库的概念，但和之前一样，我们可以选择的提供一个异步状态。
注意，所有订阅都是全局的：它们不限于 `ISubscriber` 实例的生命周期。
redis中的 发布/订阅 功能使用命名的“channels”; channels 不需要事先在服务器上定义（这里有一个有趣的用法是利用每个用户的通知渠道类驱动部分的实时更新）
如在.NET中常见的，订阅采用回调委托的形式，它接受通道名称和消息：

```C#
sub.Subscribe("messages", (channel, message) => {
    Console.WriteLine((string)message);
});
```

另外（通常在一个单独的机器上的一个单独的进程），你可以发布到该通道：

```C#
sub.Publish("messages", "hello");
```

这将（实际上瞬间）将“hello”写到订阅进程的控制台。 和之前一样，通道名和消息都可以是二进制的。

有关顺序和并发消息处理的使用文档说明，请参见 [发布/订阅消息顺序](./PubSubOrder.md) 。

访问单独的服务器
---

出于维护目的，有时需要发出服务器特定的命令：

```C#
IServer server = redis.GetServer("localhost", 6379);
```

`GetServer` 方法会接受一个 [`EndPoint`](http://msdn.microsoft.com/en-us/library/system.net.endpoint(v=vs.110).aspx) 终结点或者是一个可以唯一标识一个服务器的键/值对。

像之前介绍的那样， `GetServer` 方法返回的是一个不需要被存储的轻量级的通道对象，并且可以可选的指定 async-state（异步状态）。
需要注意的是，多个可用的节点也是可以的：

```C#
EndPoint[] endpoints = redis.GetEndPoints();
```

一个 `IServer` 实例是可以使用服务器命令的 [Server commands](http://redis.io/commands#server)，例如：

```C#
DateTime lastSave = server.LastSave();
ClientInfo[] clients = server.ClientList();
```

同步 vs 异步 vs 执行后不理
---

StackExchange.Redis有3种主要使用机制：

- 同步 - 适用于操作在方法返回到调用者之前完成（注意，尽管这可能阻止调用者，但它绝对**不会**阻止其他线程：StackExchange.Redis的关键思想是它积极地与并发调用者共享连接）

- 异步 - 操作在将来完成一些时间，并且立即返回一个 `Task` 或 'Task<T>' 对象，也可以稍后再返回：
  - 是可以等待的（阻塞当前线程，直到响应可用） `.Wait()`
  - 可以增加一个后续的回调 (TPL 中的 [`ContinueWith`](http://msdn.microsoft.com/en-us/library/system.threading.tasks.task.continuewith(v=vs.110).aspx))
  - *awaited* 可等待的（这是简化后者的语言级特性，同时如果答复已经知道也立即继续

- 执行后不理 - 适用于你真的对这个回复不感兴趣，并且乐意继续不管回应

同步的用法已经在上面的示例中展示了。这是最简单的用法，并且不涉及 [TPL][1]。

对于异步使用，关键的区别是方法名称上的 `Async` 后缀，以及（通常）使用“await”语言特性。例如：

```C#
string value = "abcdefg";
await db.StringSetAsync("mykey", value);
...
string value = await db.StringGetAsync("mykey");
Console.WriteLine(value); // writes: "abcdefg"
```

执行后不理 的用法可以通过所有方法上的可选参数 `CommandFlags flags` （默认值为 `null`）来访问使用。
这种用法中，方法会立即方法一个默认值（所以通常返回一个 `String` 的方法总是返回 `null`，而一个通常返回一个 `Int64` 的方法总是返回 `0`）。
操作会在后台继续执行。这种情况的典型用例可能是增加页面浏览数量：

```C#
db.StringIncrement(pageKey, flags: CommandFlags.FireAndForget);
```

[查看原文](https://github.com/StackExchange/StackExchange.Redis/blob/master/docs/Basics.md)
---

[1]: http://msdn.microsoft.com/en-us/library/dd460717%28v=vs.110%29.aspx
[2]: http://msdn.microsoft.com/en-us/library/system.threading.tasks.task.asyncstate(v=vs.110).aspx
[3]: http://msdn.microsoft.com/en-us/library/system.string(v=vs.110).aspx
