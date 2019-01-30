`KEYS`, `SCAN`, `FLUSHDB` 这些在哪里？
===

这里是一些非常常见的常见问题：

> 似乎没有一个 `Keys(...)` 或者 `Scan(...)` 方法？ 如何查询数据库中存在哪些键？

或者

> 似乎没有一个 `Flush(...)` 方法？如何删除数据库中的所有的键？

很奇怪的是，这里的最后一个关键词是数据库。
因为StackExchange.Redis的目标是针对集群等场景，知道哪些命令针对 *数据库* （可以是分布在多个节点上的逻辑数据库）以及哪些命令针对 *服务器* 是很重要的。
以下命令都针对单个服务器：

- `KEYS` / `SCAN`
- `FLUSHDB` / `FLUSHALL`
- `RANDOMKEY`
- `CLIENT`
- `CLUSTER`
- `CONFIG` / `INFO` / `TIME`
- `SLAVEOF`
- `SAVE` / `BGSAVE` / `LASTSAVE`
- `SCRIPT` (不要混淆 `EVAL` / `EVALSHA`)
- `SHUTDOWN`
- `SLOWLOG`
- `PUBSUB` (不要混淆 `PUBLISH` / `SUBSCRIBE` / 等)
- 一些 `DEBUG` 操作

（我可能错过了至少一个）大多数这些将显得很明显，但前3行不那么明显：

- `KEYS` / `SCAN` 只列出当前服务器上的键; 而不是更广泛的逻辑数据库
- `FLUSHDB` / `FLUSHALL` 只删除当前服务器上的密钥;而不是更广泛的逻辑数据库
- `RANDOMKEY` 仅选择当前服务器上的密钥; 而不是更广泛的逻辑数据库

实际上，StackExchange.Redis 通过简单地随机选择目标服务器来欺骗 `IDatabase` API上的 `RANDOMKEY`，但这对其他服务器是不可能的。

那么如何使用它们呢？
---

最简单的：从服务器开始，而不是数据库。

```C#
// get the target server
var server = conn.GetServer(someServer);

// show all keys in database 0 that include "foo" in their name
foreach(var key in server.Keys(pattern: "*foo*")) {
    Console.WriteLine(key);
}

// completely wipe ALL keys from database 0
server.FlushDatabase();
```

注意，与 `IDatabase` API（在 `GetDatabase()` 调用中已经选择了的目标数据库）不同，这些方法对数据库使用可选参数，或者默认为`0`。

`Keys(...)` 方法值得特别一提：它并不常见，因为它没有一个 `*Async` 对应。 这样做的原因是，在后台，系统将确定使用最合适的方法（基于服务器版本的 `KEYS` VS `SCAN`），如果可能的话，将使用 `SCAN` 方法 一个 `IEnumerable<RedisKey>` 在内部执行所有的分页 - 所以你永远不需要看到游标操作的实现细节。
如果 `SCAN` 不可用，它将使用 `KEYS`，这可能导致服务器上的阻塞。 无论哪种方式，`SCAN` 和 `KEYS` 都需要扫描整个键空间，所以在生产服务器上应该避免 - 或者至少是针对从节点服务器。

所以我需要记住我连接到哪个服务器？ 这真糟糕！
---

不，不完全是。 你可以使用 `conn.GetEndPoints()` 来列出节点（所有已知的节点，或者在原始配置中指定的节点，这些不一定是相同的东西），并且使用 `GetServer()` 迭代找到想要的服务器（例如，选择一个从节点）。

[查看原文](https://github.com/StackExchange/StackExchange.Redis/blob/master/docs/KeysScan.md)
---

[返回主页](./README.md)
---