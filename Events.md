事件
===

`ConnectionMultiplexer` 类型提供了许多事件可以用来理解被封装的底层是怎么工作的。这在记录日志时会特别有用。

- `ConfigurationChanged` - 当连接的配置从 `ConnectionMultiplexer` 内部发生修改时触发
- `ConfigurationChangedBroadcast` - 当经由发布/订阅接收到重新配置消息时引发; 这通常是由于 `IServer.MakeMaster` 用于更改节点的复制配置，可以选择将这样的请求广播到所有客户端
- `ConnectionFailed` - 当连接由于无论任何原因失败时触发; 请注意，在连接重新建立之前是不会再收到该连接的 `ConnectionFailed` 通知
- `ConnectionRestored` - 当重新建立到先前失败的节点的连接时触发
- `ErrorMessage` - 当redis服务器响应任何用户发起的具有错误消息的请求时触发; 这种情况不包含将被报告给直接调用者的常规异常/故障的情况
- `HashSlotMoved` - 当“redis集群”指出 散列槽（hash-slot） 已在节点之间迁移时触发; 请注意，请求通常会自动重新路由，因此用户不需要在这里做任何特殊操作
- `InternalError` - 当库在一些意想不到的方式失败时触发; 这主要是为了调试目的，并且大多数用户应该不需要这个事件

需要注意，StackExchange.Redis 中的 发布/订阅 工作方式与事件*非常相似*，接收到消息时会调用接受一个 `Action<RedisChannel, RedisValue>` 类型回调方法的 `Subscribe` / `SubscribeAsync` 方法。