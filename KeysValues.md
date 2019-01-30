键，值和通道
===

在处理redis时，是键不是键之间有很重要的区别。
键是数据库中数据片段（可以是String，List，Hash或任何其他[redis数据类型](http://redis.io/topics/data-types)）的独一无二的名称。
键永远不会被解释为...好吧，任何东西：它们只是惰性名称。
此外，当处理集群或分片系统时，它是定义包含此数据的节点（或者如果有从节点的节点）的关键 - 因此键对于路由命令是至关重要的。

这与 *值* 形成对比； 值是单独（对于字符串数据）或分组对键的*内容存储*  。
值不影响命令路由 <small>（注意：使用 SORT 命令时除非指定 BY 或者 GET，否则是很难解释的）</small>

同样，为了操作的目的，值通常被redis翻译为：

- `incr` （和各种类似的命令）将String值转换为数值数据
- 排序可以使用数字或unicode规则解释值
- 和许多其他操作

关键是使用API需要理解什么是键，什么是值。 
这反映在StackExchange.Redis API中，但是好消息是，**大部分时间**你根本不需要知道这一点。

当使用 发布/订阅 时，我们处理 *channels* ; channel 不会影响路由（因此它们不是密钥），但与常规值非常不同，因此要单考虑。

键
---

StackExchange.Redis 通过 `RedisKey` 类型表示键。
好消息是，可以从 `string` 和 `byte[]` 的隐式转换，允许使用文本和二进制密钥，没有任何复杂性。

例如，`StringIncrement` 方法使用一个 `RedisKey` 作为第一个参数，但是*你不需要知道* ; 

举个例子：

```C#
string key = ...
db.StringIncrement(key);
```

or

```C#
byte[] key = ...
db.StringIncrement(key);
```

同样，有一些操作*返回* 键为 `RedisKey` - 再次，它依然可以自动隐式转换：

```C#
string someKey = db.KeyRandom();
```

值
---

StackExchange.Redis 用 `RedisValue` 类型表示值。 与 `RedisKey` 一样，存在隐式转换，这意味着大多数时候你从来没有看到这种类型，例如：

```C#
db.StringSet("mykey", "myvalue");
```

然而，除了文本和二进制内容，值还可能需要表示类型化的原始数据 - 最常见的（在.NET术语中）`Int32`，`Int64`，`Double`或`Boolean`。 因此，`RedisValue`提供了比 `RedisKey` 更多的转换支持：

```C#
db.StringSet("mykey", 123); // this is still a RedisKey and RedisValue
...
int i = (int)db.StringGet("mykey");
```

请注意，虽然从基元类型到 `RedisValue` 的转换是隐式的，但是从 `RedisValue` 到基元类型的许多转换是显式的：这是因为如果数据没有合适的值，这些转换很可能会失败。

另外注意，*当做数字* 处理时，redis将不存在的键视为零; 为了与此一致，将空响应视为零：

```C#
db.KeyDelete("abc");
int i = (int)db.StringGet("abc"); // this is ZERO
```

如果您需要检测空状态，那么你就可以这样检查：

```C#
db.KeyDelete("abc");
var value = db.StringGet("abc");
bool isNil = value.IsNull; // this is true
```

或者更简单地，只是使用提供的 `Nullable <T>` 支持：

```C#
db.KeyDelete("abc");
var value = (int?)db.StringGet("abc"); // behaves as you would expect
```

哈希
---

由于哈希中的字段名称不影响命令路由，它们不是键，但可以接受文本和二进制名称， 因此它们被视为用于API目的的值。

通道
---

发布/订阅 的通道名称由 `RedisChannel` 类型表示; 这与 `RedisKey` 大体相同，但是是独立处理的，因为虽然通道名是正当的第一类元素，但它们不影响命令路由。

脚本
---

[redis中的脚本](http://redis.io/commands/EVAL) 有两项显著的特性：

- 输入必须保持键和值分离（在脚本内部分别成为 `KEYS` 和 `ARGV`）
- 返回格式未预先定义：这将特定于您的脚本

正因为如此，`ScriptEvaluate` 方法接受两个独立的输入数组：一个用于键的 `RedisKey []`，一个用于值的 `RedisValue []` （两者都是可选的，如果省略则假定为空）。 这可能是你实际需要在代码中键入 `RedisKey` 或 `RedisValue` 的少数几次之一，这只是因为数组变动规则：

```C#
    var result = db.ScriptEvaluate(TransferScript,
    new RedisKey[] { from, to }, new RedisValue[] { quantity });
```

（其中 `TransferScript` 是一些包含Lua的 `string`，在这个例子中没有显示）

响应使用 `RedisResult` 类型（这是脚本专用的;通常API尝试尽可能直接清晰地表示响应）。 和前面一样， `RedisResult` 提供了一系列转换操作 - 实际上比 `RedisValue` 更多，因为除了可以转换为文本，二进制，一些基元类型和可空元素，响应*也*可以转换为 *数组* ，例如：

```C#
string[] items = db.ScriptEvaluate(...);
```

结论
---

API中使用的类型是非常故意选择的，以区分redis *keys* 和 *values*。 然而，在几乎所有情况下，您不需要直接去参考所涉及的底层类型，因为提供了转换操作。

[查看原文](https://github.com/StackExchange/StackExchange.Redis/blob/master/docs/KeysValues.md)
---

[返回主页](./README.md)
---