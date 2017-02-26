脚本
===

`IServer.ScriptLoad(Async)`、 `IServer.ScriptExists(Async)`、`IServer.ScriptFlush(Async)`、 `IDatabase.ScriptEvaluate` 和 `IDatabaseAsync.ScriptEvaluateAsync` 这些方法为基本的 [Lua脚本](http://redis.io/commands/EVAL) 提供了支持。
这些方法暴露了向Redis提交和执行Lua脚本所需的基本命令。

通过 `LuaScript` 类可以获得更复杂的脚本。 `LuaScript` 类使得更容易准备和提交参数以及脚本，以及允许您使用清理代码后变量名称。

`LuaScript` 的使用示例:

```
	const string Script = "redis.call('set', @key, @value)";

	using (ConnectionMultiplexer conn = /* init code */)
	{
		var db = conn.GetDatabase(0);

		var prepared = LuaScript.Prepare(Script);
		db.ScriptEvaluate(prepared, new { key = (RedisKey)"mykey", value = 123 });
	}
```

`LuaScript` 类将 `@myVar` 形式的脚本中的变量重写为redis所需的合适的 `ARGV [someIndex]`。 
如果传递的参数是 `RedisKey` 类型，它将作为 `KEYS` 集合的一部分自动发送。

Any object that exposes field or property members with the same name as @-prefixed variables in the Lua script can be used as a parameter hash to
`Evaluate` calls.  

任何在Lua脚本中暴露的以`@`为前缀变量同名的字段或属性成员的对象都可以用作参数哈希 `Evaluate` 调用。

支持的成员类型如下：

 - int(?)
 - long(?)
 - double(?)
 - string
 - byte[]
 - bool(?)
 - RedisKey
 - RedisValue

为了避免在每次评估时重新传输Lua脚本到redis，`LuaScript` 对象可以通过 `LuaScript.Load(IServer)` 转换为 `LoadedLuaScript`。
`LoadedLuaScripts` 使用 [`EVALSHA`](http://redis.io/commands/evalsha) 求值，并由 hash 引用。

`LoadedLuaScript` 的使用示例：

```
	const string Script = "redis.call('set', @key, @value)";

	using (ConnectionMultiplexer conn = /* init code */)
	{
		var db = conn.GetDatabase(0);
		var server = conn.GetServer(/* appropriate parameters*/);

		var prepared = LuaScript.Prepare(Script);
		var loaded = prepared.Load(server);
		loaded.Evaluate(db, new { key = (RedisKey)"mykey", value = 123 });
	}
```

`LuaScript` 和 `LoadedLuaScript` 上的所有方法都有Async替代方法，并将提交到redis的实际脚本公开为 `ExecutableScript` 属性。

[查看原文](https://github.com/StackExchange/StackExchange.Redis/blob/master/Docs/Scripting.md)
---