基本使用
===

StackExchange.Redis 最重要的对象是由`StackExchange.Redis`这个名称空间里面的`ConnectionMultiplexer`类产生的；这个对象隐藏了许多细节。因为 `ConnectionMultiplexer`实现了很多，它被设计用来在调用者之间**共享和复用**。所以，你没有必要在每次操作中都创建一个`ConnectionMultiplexer`的对象。 而它完全是线程安全，且有值被调用的。在随后的所有示例中，我们都假设你已经保存了一个可被复用的`ConnectionMultiplexer`实例。但现在，我们先创建一个吧。通过使用`ConnectionMultiplexer.Connect`或`ConnectionMultiplexer.ConnectAsync`，并传递一个配置字串，或`ConfigurationOptions`类型对象，它就被创建好了。配置字串可以是由一系列逗号分隔的节点集，那么现在，我们就来连接Redis本地的默认端口（6379）:

备注：

StackExchange.Redis ：Redis的堆栈交换

ConnectionMultiplexer  ： 多路连接

```C#
using StackExchange.Redis;
...
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("localhost");
// ^^^ store and re-use this!!!
```

注意，当`ConnectionMultiplexer`不再需要的时候就会被释放，因为`ConnectionMultiplexer`实现了`IDisposable`。这里有意不使用声明的用法，因为目的是复用这个对象，所以你会极少地想要使用一个更简洁的`ConnectionMultiplexer`。

解决一个主/从服务的构建将会是一个更加复杂的方案；为了实现该方案，这里只是简单地规定所有被要求的节点都能作为 redis 的逻辑数据层（它将自动地识别主服务）：

```C#
ConnectionMultiplexer redis = ConnectionMultiplexer.Connect("server1:6379,server2:6379");
```

如果它找到所有的节点都是主服务，那么 tie-breaker 键将会选择性的被指定，并且以此来解决这个问题。但是无论如何，这种情况是非常少的。

一旦你创建了一个`ConnectionMultiplexer`的实例，那么你将会有3个主要事情去做：

- 连接 redis 的数据库（要注意到，就集群来说，单个逻辑上的数据库可能会分布在多个数据库节点之间）
- 使用 redis 的[发送/订阅](http://redis.io/topics/pubsub)特性
- 为维护/监控的目的去连接一个私人的服务器

使用 redis 数据库
---

如此简单地连接一个 redis 的数据库：

```C#
IDatabase db = redis.GetDatabase();
```

从`GetDatabase`方法返回的对象将会是一个直通、开销较少的对象，所以并不需要被存储起来。要注意到， redis 提供了多种多样的服务器（虽然这个并不是集群）；所以我们就可以选择性地指明调用`GetDatabase`的方法。另外，如果你想要使用异步的API，并且你想要给[`Task.AsyncState`][2]赋值，那么同样需要指明：

```C#
int databaseNumber = ...
object asyncState = ...
IDatabase db = redis.GetDatabase(databaseNumber, asyncState);
```

如果你使用了`IDatabase`，也不过是简单地使用了[redis API](http://redis.io/commands)的一种情况。我们注意到，所有的方法要么是异步的方法，要么是实现了异步接口的方法。为了和微软的命名规范达成一致，我们所有的异步方法都是以`...Async(...)`结尾的，而且都是可以等待（`await`-able）的哦。

存值，取值的最简单的操作：

```C#
string value = "abcdefg";
db.StringSet("mykey", value);
...
string value = db.StringGet("mykey");
Console.WriteLine(value); // writes: "abcdefg"
```

我们注意到，这个`String...`的前缀表示了 redis 的 [String 类型](http://redis.io/topics/data-types)，虽然都可以被用来保存文本数据，但是这个类型是与[.NET的String类型][3]不同的。那么 redis 是允许将未处理的二进制数据作为它的键和值来使用的：

```C#
byte[] key = ..., value = ...;
db.StringSet(key, value);
...
byte[] value = db.StringGet(key);
```

所有的[redis database commands](http://redis.io/commands)（redis 数据库指令） 中涉及到的数据类型都是可以被使用的。

使用 redis 发布/订阅
----

通常另一个 redis 的使用，就是作为一个消息[发布/订阅](http://redis.io/topics/pubsub)的工具；同样很简单，在失败连接事件中，这个`ConnectionMultiplexer`将会处理重新订阅对应频道的一些细节。

```C#
ISubscriber sub = redis.GetSubscriber();
```

同样地，从`GetSubscriber`方法返回的是一个低开销的，直通的对象，并不需要被存储下来。发布/订阅 API 没有数据库设计的理念，但是就像之前一样，我们可以选择性地提供一个异步的状态。我们注意到所有的订阅都是全局的：它们并没有在`ISubscriber`这个对象实例的生命周期内被弄明白。在 redis 里面，发布/订阅的特性将使用“频道”来命名；而频道不需要预先在服务器中被定义（就好像每个用户都关心一个通知的频道，它们将非常有趣，这也是[Stack Overflow](http://stackoverflow.com))能够实时更新的道理）：

```C#
sub.Subscribe("messages", (channel, message) => {
    Console.WriteLine((string)message);
});
```

你可以分布式地（通常在一个分布式服务器的进程里）向这个频道发布消息：

```C#
sub.Publish("messages", "hello");
```

这将会（几乎瞬间）向订阅进程的控制台中去写入"hello"。就像之前一样，频道名字和消息都可以被二进制序列化。

也请看看[Pub / Sub Message Order](PubSubOrder)对顺序而不是并发进程的介绍。

连接个人的服务器
---

为了便于维护，有时候需要去发布服务端程序特有的命令：

```C#
IServer server = redis.GetServer("localhost", 6379);
```

这个`GetServer`方法也可以接受一个[`EndPoint`](http://msdn.microsoft.com/en-us/library/system.net.endpoint(v=vs.110).aspx)（标识了网络地址）类型，或者唯一标识服务器的名字/值对的参数。就像之前一样，从 `GetServer `返回的是一个没必要存储的低消耗、直通的对象，并且异步的状态是需要被有选择指明的。我们注意到，必要的endpoints（标识了网络地址）设置是同样需要的：

```C#
EndPoint[] endpoints = redis.GetEndPoints();
```

从 IServer  实例来分析，[Server commands](http://redis.io/commands#server)（服务器指令）是同样必须的；举个例子：

```C#
DateTime lastSave = server.LastSave();
ClientInfo[] clients = server.ClientList();
```

同步 vs 异步 vs Fire-and-Forget
---

使用 StackExchange.Redis 有3个主要的使用机制：

- 同步性 - 表现在方法返回给调用者之前，操作已经完成了（我们注意到，这个操作可能会阻塞调用者，但是它绝对**不会**阻塞其他的线程：关键是在 StackExchange.Redis 里面将主动分享 connection （连接）给同时调用者）
- 异步性 - 表现在有时候完成操作会有延迟，但是`Task`或者`Task<T>`是会被立即返回的，这种情况会有延迟：
 - 以`.Wait()`结尾的（将阻塞当前的线程，直到被响应）
 - 使用一个连续的回调，需要附加（使用TPL中的[`ContinueWith`](http://msdn.microsoft.com/en-us/library/system.threading.tasks.task.continuewith(v=vs.110).aspx)）
 - 能够被*等待*（可以简化的后者是一个语言级别的特性，如果回复已经知道了，就会立即继续）
- Fire-and-Forget - 表现在你真的对回应不感兴趣，并且也不乐意考虑这个回应

同步的使用已经在上面的例子演示了。这是一个最简单的使用，并不需要 [TPL][1]。

作为异步的使用，关键的是方法后缀`Async`上不同，（典型地）使用`await`的语言特性。举个例子：

```C#
string value = "abcdefg";
await db.StringSetAsync("mykey", value);
...
string value = await db.StringGetAsync("mykey");
Console.WriteLine(value); // writes: "abcdefg"
```

fire-and-forget 的使用是在所有方法上都使用`CommandFlags`的选项（默认是没有的）。在这种用法中，方法会立即返回默认的值（那么，如果是`String`类型，方法通常会返回`null`；如果是`Int64`类型，会返回`0`）。操作会在后台继续。一个典型的使用，可能会是 page-view 计数的自增：

```C#
db.StringIncrement(pageKey, flags: CommandFlags.FireAndForget);
```




  [1]: http://msdn.microsoft.com/en-us/library/dd460717%28v=vs.110%29.aspx
  [2]: http://msdn.microsoft.com/en-us/library/system.threading.tasks.task.asyncstate(v=vs.110).aspx
  [3]: http://msdn.microsoft.com/en-us/library/system.string(v=vs.110).aspx
