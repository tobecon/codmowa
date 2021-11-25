### 事件现象
    在进行 公司 asp.net core 项目开发的时候, 出现一个有趣的bug, 有的接口返回的 datetime 类型为后缀带Z eg `2021-11-24T12:31:50.2246113Z` 的 UTC时间, 有的接口返回的是 正常的 YYYY:MM:DD HH:mm:ss eg:2021-11-24 20:31:56. 这种现象导致前端进行适配的时候 非常困难.
    但是因为业务中对时分秒的要求没有太高, 所以这个现象一直没有引起太大的注意, 直到最近开始注意到这个现象
### 调查过程
    - 一开始很自然的以为是 Newtonsoft.json  或者 System.Text.json 序列化的问题. 因为是公司封装了一个 Controller base类型. 替换了 asp.net core ActionResult. 但是经过几番排查和试错发现这种现象还是存在. 事情进入死胡同
    - 然后就发现经过ef core 加载出来的数据的 Datetime 类型的 DatetimeKind 是Utc. 这里应该是local 才对. 然后就怀疑是不是 ef core 和 MSSQL 交互的时候出现了什么问题.
    - 但是翻了翻 efcore 的源码 发现没有什么地方 明确改变了Datetime 的Kind
    - 于是只能去看这个接口的实现代码, 发现有一层缓存. 那么很自然的怀疑到这一层缓存上面.
    - 公司缓存数据 用的是 MessagePack 做格式化, 和 redis做存储. 
    - 那么嫌疑人只能是 MessagePack 和 redis了 
### 测试代码
```c#
    [Route("type")]
    [ApiController]
    public class MyDatetimeController : AmiController
    {
        private readonly ICache cache;

        public MyDatetimeController(ICache cache)
        {
            this.cache = cache;
        }
        [Route("no-cache")]
        public async Task<AmiResult> Get()
        {
            return Ok(new MyDatetime { Mydate = DateTime.Now, MydateUtc = DateTime.UtcNow });
        }
        [Route("cache")]
        public async Task<AmiResult> GetCache()
        {
            var m = new MyDatetime { Mydate = DateTime.Now, MydateUtc = DateTime.UtcNow };
            cache.Cache(CacheKey.DV, "testd", m);
            cache.TryGetValue(CacheKey.DV, "testd", out m);
            return Ok(m);
        }
    }
    [MessagePack.MessagePackObject]
    public class MyDatetime
    {
        [MessagePack.Key(1)]
        public DateTime MydateUtc { get; set; }
        [MessagePack.Key(2)]
        public DateTime Mydate { get; set; }
    }
```
### 测试结果
```json

http://localhost:8011/amiapicore/api/type/no-cache  
{
  "content": {
    "mydateUtc": "2021-11-24T12:31:56.1622685Z",
    "mydate": "2021-11-24T20:31:56.1622681+08:00"
  },
  "errorCode": 0
}

http://localhost:8011/amiapicore/api/type/cache
{
  "content": {
    "mydateUtc": "2021-11-24T12:31:50.2246113Z",
    "mydate": "2021-11-24T12:31:50.2245647Z"
  },
  "errorCode": 0
}
```

### 结论
    - 很自然的看出, 是MessagePack 在序列化的时候有点不一样,事实也确实如此 找到了 MessagePack的官方文档, 上面写着
```
DateTime is serialized to MessagePack Timestamp format, it serialize/deserialize UTC and loses Kind info and requires that MessagePackWriter.OldSpec == false. If you use the NativeDateTimeResolver, DateTime values will be serialized using .NET's native Int64 representation, which preserves Kind info but may not be interoperable with non-.NET platforms.
```
    - 简单翻译一下就是 DatetTime 被序列化成了 MessagePack Timestamp 格式, 并去掉了 Datetime 的 Kind 信息 以便和其他的系统交流.
    - 如果你想保留Kind 信息,只能使用 NativeDateTimeResolver, 但是这样就只能与 .Net 平台交互.
```
### 解决方案

```c#
    [MessagePack.MessagePackObject]
    public class MyDatetime
    {
        [MessagePack.Key(1)]
        public DateTime MydateUtc { get; set; }
        // 简单的话就加上下面的Atrribute, 让MessagePack 先不做格式转换, 复杂的话就要和公司的业务结合起来考虑了;
        [MessagePack.Key(2)]
        [MessagePack.MessagePackFormatter(typeof(MessagePack.Formatters.NativeDateTimeFormatter))]
        public DateTime Mydate { get; set; }
    }

    // or 
    public static class Bson
    {

        private static MessagePack.IFormatterResolver CreateResolver()
        {
            var resolver = MessagePack.Resolvers.CompositeResolver.Create(    
            NativeDateTimeResolver.Instance,
            StandardResolverAllowPrivate.Instance);
            return resolver;
        }
        private static MessagePackSerializerOptions Options = MessagePackSerializerOptions.Standard.WithResolver(CreateResolver());
        public static byte[] Serialize<T>(T data)
        {
            return MessagePackSerializer.Serialize<T>(data,Options);
        }

        public static T Deserialize<T>(byte[] data)
        {
            return MessagePackSerializer.Deserialize<T>(data, Options);
        }
    }
```



### 引用

[MessagePack-Csharp](https://github.com/neuecc/MessagePack-CSharp#:~:text=DateTime%20is%20serialized%20to,with%20non%2D.NET%20platforms.)
[MessagePack-Solution](https://github.com/neuecc/MessagePack-CSharp#:~:text=CompositeResolver%20as%20shown%20below%3A)

