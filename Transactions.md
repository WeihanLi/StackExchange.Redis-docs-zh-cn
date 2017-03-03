Redis中的事务
=====================

Redis中的事务不像SQL数据库中的事务。
[完整的文档在这里](http://redis.io/topics/transactions)，这里稍微借用一下：

redis中的事务包括放置在 `MULTI` 和 `EXEC` 之间的一组命令（或者用于回滚的 `DISCARD`）。
一旦遇到 `MULTI`，该连接相关的命令*不会被执行* - 它们会进入一个队列（并且 每一个命令调用者得到一个答复 `QUEUED`）。

当遇到一个 `EXEC` 时，它们都被应用在一个单元中（即在操作期间没有其他连接获得时间去做任何事）。
如果看到 `DISCARD` 而不是 `EXEC`，则一切都被抛弃。因为事务中的命令会排队，你不能在事务*里面* 操作。

例如，在SQL数据库中，您可以执行以下操作（伪代码 - 仅供说明）：

```C#
// assign a new unique id only if they don't already
// have one, in a transaction to ensure no thread-races
var newId = CreateNewUniqueID(); // optimistic
using(var tran = conn.BeginTran())
{
	var cust = GetCustomer(conn, custId, tran);
	var uniqueId = cust.UniqueID;
	if(uniqueId == null)
	{
		cust.UniqueId = newId;
		SaveCustomer(conn, cust, tran);
	}
	tran.Complete();
}
```

如何在Redis中实现事务呢？
---

这在redis事务中是不可能的：一旦事务被打开你*不能获取数据* - 你的操作被排队。 幸运的是，还有另外两个命令帮助我们： `WATCH` 和 `UNWATCH` 。

`WATCH {key}` 告诉Redis，我们对用于事务目的的特定的键感兴趣。
Redis会自动跟踪这个键，任何变化基本上都会使我们的事务回滚 - `EXEC` 和 `DISCARD` 一样（调用者可以检测到这一点，并从头开始重试）。
所以你可以做的是： `WATCH` 一个键，以正常的方式检查该键的数据，然后 `MULTI` / `EXEC` 你的更改。

如果，当你检查数据，你发现你实际上不需要事务，你可以使用 `UNWATCH` 来取消关注所有关注的键。 
注意，关注的键在 `EXEC` 和 `DISCARD` 期间也被复位。 所以*在Redis层*，事务是从概念上讲的。

```
WATCH {custKey}
HEXISTS {custKey} "UniqueId"
(check the reply, then either:)
MULTI
HSET {custKey} "UniqueId" {newId}
EXEC
(or, if we find there was already an unique-id:)
UNWATCH
```

这可能看起来很奇怪 - 有一个 `MULTI` / `EXEC` 只跨越一个操作 - 但重要的是，我们现在也从其他所有连接跟踪对 `{custKey}` 的更改 - 如果任何人更改键 ，事务将被中止。

在 StackExchange.Redis 中如何实现事务？
---

说实话，StackExchange.Redis 使用多路复用器方法实现事务更复杂。
我们不能简单地让并发的调用者发出 `WATCH` / `UNWATCH` / `MULTI` / `EXEC` / `DISCARD`：它会全部混在一起。

因此，StackExchange.Redis 提供了额外的抽象来使事情更简单的变得正常：*constraints*。

*Constraints*  基本上是预先测试涉及 `WATCH` ，一些测试，以及结果的检查。 
如果所有约束都通过，则触发 `MULTI` / `EXEC` ， 否则触发 `UNWATCH` 。

这是以防止与其他调用者混合在一起的命令的方式完成的。 所以我们的例子变成了：

```C#
var newId = CreateNewId();
var tran = db.CreateTransaction();
tran.AddCondition(Condition.HashNotExists(custKey, "UniqueID"));
tran.HashSetAsync(custKey, "UniqueID", newId);
bool committed = tran.Execute();
// ^^^ if true: it was applied; if false: it was rolled back
```

注意，从 `CreateTransaction` 返回的对象只能访问 *async* 方法 - 因为每个操作的结果在 `Execute`（或 `ExecuteAsync` ）完成之前都不会知道。 
如果操作不应用，所有的任务将被标记为已取消 - 否则，命令执行*后*，您可以正常获取每个的结果。

可用*条件* 的集合不是广泛的，而是涵盖最常见的情况；如果你还想看到其他条件，请与我联系（或更好的方式：提交 pull-request）。

借助 `When` 的内置操作
---

还应该注意的是，Redis 预期已经了许多常见的情况（特别是：密钥/散列 存在，如上所述），所以存在单次操作原子命令。

这些是通过`When`参数访问的 - 所以我们前面的例子可以也可以写成：

```C#
var newId = CreateNewId();
bool wasSet = db.HashSet(custKey, "UniqueID", newId, When.NotExists);
```

（这里，`When.NotExists` 导致使用 `HSETNX` 命令，而不是 `HSET`）

Lua 脚本
---

你还应该知道，Redis 2.6及以上版本[支持Lua脚本](http://redis.io/commands/EVAL)，用于在服务器端执行多个作为单个原子单元的操作的通用工具。由于在Lua脚本中没有服务于其他连接，它的行为很像一个事务，但没有 `MULTI` / `EXEC` 等这样复杂。
这也避免了在调用者和服务器之间的带宽和延迟等问题
，但是需要与脚本垄断服务器的持续时间之间权衡。

在Redis层（假设 `HSETNX` 不存在），这可以实现为：

```
EVAL "if redis.call('hexists', KEYS[1], 'UniqueId') then return redis.call('hset', KEYS[1], 'UniqueId', ARGV[1]) else return 0 end" 1 {custKey} {newId}
```

这可以在 StackExchange.Redis 中使用：

```C#
var wasSet = (bool) db.ScriptEvaluate(@"if redis.call('hexists', KEYS[1], 'UniqueId') then return redis.call('hset', KEYS[1], 'UniqueId', ARGV[1]) else return 0 end",
        new RedisKey[] { custKey }, new RedisValue[] { newId });
```

（注意 `ScriptEvaluate` 和 `ScriptEvaluateAsync` 的响应是可变的，这取决于你确切的脚本，响应可以被强制中断 - 在这种情况下为 `bool`）

[查看原文](https://github.com/StackExchange/StackExchange.Redis/blob/master/Docs/Transactions.md)
---