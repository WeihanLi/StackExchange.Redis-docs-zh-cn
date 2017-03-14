管道和链接复用
===

延迟严重。现代计算机可以以惊人的速度搅动数据，并且高速网络（通常具有在重要服务器之间的多个并行链路）提供巨大的带宽，但是... 该延迟意味着计算机花费大量的时间*等待数据* 和 这是基于连续的编程越来越受欢迎的几个原因之一。


让我们考虑一些常规的程序代码：

```C#
string a = db.StringGet("a");
string b = db.StringGet("b");
```

在涉及的步骤方面，这看起来大致是这样：

    [req1]                         # client: the client library constructs request 1
         [c=>s]                    # network: request one is sent to the server
              [server]             # server: the server processes request 1
                     [s=>c]        # network: response one is sent back to the client
                          [resp1]  # client: the client library parses response 1
                                [req2]
                                     [c=>s]
                                          [server]
                                                 [s=>c]
                                                      [resp2]

现在让我们突出*客户端正在做的一些事的时间点*：

    [req1]
         [====waiting=====]
                          [resp1]
                                [req2]
                                     [====waiting=====]
                                                      [resp2]

请记住，这是**不成比例的** - 如果这是按时间缩放，它将是完全由` waiting`  控制的。

管道线
---

因此，许多redis客户端允许你使用 *pipelining*， 这是在管道上发送多个消息而不等待来自每个的答复的过程 - 并且（通常）稍后当它们进入时处理答复。
在 .NET 中，可以启动但尚未完成，并且可能以后完成或故障的由TPL封装的操作可以由 [TPL][1] 通过 [`Task`][2] / [`Task <T>`][3] API 来实现。
本质上， `Task<T>` 表示 " `T` 类型未来可能的值"（非泛型的 `Task` 本质尚是 `Task<void>` ）。你可以使用任意一种用法：

-  在稍后的代码块等待直到操作完成（`.Wait()`）
- 当操作完成时，调度一个后续操作（`.ContinueWith(...)` 或 `await`）

例如，要使用过程化（阻塞）代码来借助管道传递这两个 get 操作，我们可以使用：

```C#
var aPending = db.StringGetAsync("a");
var bPending = db.StringGetAsync("b");
var a = db.Wait(aPending);
var b = db.Wait(bPending);
```

注意，我在这里使用`db.Wait`，因为它会自动应用配置的同步超时，但如果你喜欢你也可以使用 `aPending.Wait()` 或 `Task.WaitAll(aPending,bPending);`。
使用管道技术，我们可以立即将这两个请求发送到网络，从而消除大部分延迟。
此外，它还有助于减少数据包碎片：单独发送（等待每个响应）的20个请求将需要至少20个数据包，但是在管道中发送的20个请求可以通过少得多的数据包（也许只有一个）。

执行后不理
---

管道的一个特例是当我们明确地不关心来自特定操作的响应时，这允许我们的代码在排队操作在后台继续时立即继续。 通常，这意味着我们可以在单个调用者的连接上并发工作。 这是使用 `flags` 参数来实现的：

```C#
// sliding expiration
db.KeyExpire(key, TimeSpan.FromMinutes(5), flags: CommandFlags.FireAndForget);
var value = (string)db.StringGet(key);
```

`FireAndForget` 标志使客户端库正常地排队工作，但立即返回一个默认值（因为 `KeyExpire` 返回一个 `bool` ，这将返回 `false` ，因为 `default(bool)` 是 `false`  - 但是返回值是无意义的，应该忽略）。
这也适用于 `*Async` 方法：一个已经完成的 `Task <T>` 返回默认值（或者为 `void` 方法返回一个已经完成的 `Task` ）。

复用链接
---

管道是很好的，但是通常任何单个代码块只需要一个值（或者可能想要执行几个操作，但是依赖于彼此）。
这意味着我们仍然有一个问题，我们花大部分时间等待数据在客户端和服务器之间传输。 
现在考虑一个繁忙的应用程序，也许是一个Web服务器。 
这样的应用程序通常是并发的，所以如果你有20个并行应用程序请求都需要数据，你可能会想到旋转20个连接，或者你可以同步访问单个连接（这意味着最后一个调用者需要等待 延迟的所有其他19之前，甚至开始）。 或者折中一下，也许一个5个连接的租赁池 - 无论你怎么做，都会有很多的等待。
**StackExchange.Redis不做这个**; 相反，它做 *很多* 的工作，使你有效地利用所有这个空闲时间*复用* 一个连接。
当不同的调用者同时使用它时，它**自动把这些单独的请求加入管道**，所以不管请求使用阻塞还是异步访问，工作都是按进入管道的顺序处理的。
因此，我们可能有10或20个的“get a 和 b”包括此前的（从不同的应用程序请求）情景中，并且他们都将尽快到达连接。
基本上，它用完成其他调用者的工作的时间来填充 `waiting` 时间。

因此，StackExchange.Redis不提供的唯一redis特性（*不会提供*）是“阻塞弹出”（[BLPOP](http://redis.io/commands/blpop)，[BRPOP](http://redis.io/commands/brpop) 和 [BRPOPLPUSH](http://redis.io/commands/brpoplpush)），因为这将允许单个调用者停止整个多路复用器，阻止所有其他调用者 。
StackExchange.Redis 需要保持工作的唯一其他时间是在验证事务的前提条件时，这就是为什么 StackExchange.Redis 将这些条件封装到内部管理的 `condition` 实例中。

[在这里阅读更多关于事务](https://github.com/StackExchange/StackExchange.Redis/blob/master/Docs/Transactions.md)。
如果你觉得你想“阻止出栈”，那么我强烈建议你考虑 发布 / 订阅 代替：

```C#
sub.Subscribe(channel, delegate {
    string work = db.ListRightPop(key);
    if (work != null) Process(work);
});
//...
db.ListLeftPush(key, newWork, flags: CommandFlags.FireAndForget);
sub.Publish(channel, "");
```

这实现了相同的目的，而不需要阻塞操作。 注意：

- *数据* 不通过 发布 / 订阅 发送; 发布 / 订阅 API只用于通知处理器检查更多的工作
- 如果没有处理器，则新项目保持缓存在列表中; 工作不会丢失
- 只有一个处理器可以弹出单个值; 当消费者比生产者多时，一些消费者会被通知，然后发现没有什么可做的
- 当你重新启动一个处理器，你应该*假设* 有工作，以便你处理任何积压的任务
- 但除此之外，语义与阻止出栈相同

StackExchange.Redis 的多路复用特性使得在使用常规的不复杂代码的同时在单个连接上达到极高的吞吐量成为可能。

并发
---

应当注意，管道/链接复用器/未来值 方法对于基于连续的异步代码也很好地起作用；例如你可以写：

```C#
string value = await db.StringGetAsync(key);
if (value == null) {
    value = await ComputeValueFromDatabase(...);
    db.StringSet(key, value, flags: CommandFlags.FireAndForget);
}
return value;
```

[查看原文](https://github.com/StackExchange/StackExchange.Redis/blob/master/Docs/PipelinesMultiplexers.md)
---

  [1]: http://msdn.microsoft.com/en-us/library/dd460717(v=vs.110).aspx
  [2]: http://msdn.microsoft.com/en-us/library/system.threading.tasks.task(v=vs.110).aspx
  [3]: http://msdn.microsoft.com/en-us/library/dd321424(v=vs.110).aspx
