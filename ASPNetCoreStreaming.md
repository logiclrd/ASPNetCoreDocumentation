# Streaming Responses in ASP.NET Core

Some web endpoints need to transfer a large amount of data to the client, such that buffering that data in memory would be impractical or impossible. In some cases, the data might be of indeterminate length, such as a live video stream, which needs to continue transmitting data as long as a client remains connected. Most aspects of ASP.NET Core are oriented toward HTTP requests that are short and complete quickly, and this is what the web protocol and server & client software is optimized for, but for some use cases, there is no getting around returning large blocks of data. It is important to design the code for such endpoints carefully, to ensure that they do not consume too many resources and that they behave properly in all cases. If problems arise, they can affect the functionality of the entire site, and, indirectly, even other resources on the same server.

The common path offered by ASP.NET Core is designed to provide the best balance of simplicity and performance for short, small requests. There are patterns that can provide effectively for delivering data without keeping it in memory all at once. Some aspects require care to ensure that they are implemented correctly.

In this topic, the term _streaming data_ is used to describe a situation where the quantity of data is either large enough that it is important to avoid buffering, or it is important to deliver content to the client immediately as it becomes available, rather than returning the entire response as a unit.

## Conventions

The sample code presented in this article assumes that you are using Dependency Injection with your ASP.NET Core service implementation. As such, when classes have parameters on the constructor and expect to be handed objects of the types they need, this is handled by means of the D.I. collection configured by the `ConfigureServices` method in the `Startup` class.

It is also assumed that types will typically be exposed by means of interfaces. This is a common practice that enables "mock" objects to be injected into instances to allow automated testing, which can greatly improve code quality. You may adapt the examples here to your project's conventions as needed.

## Structured Data with System.Text.Json

If you need to stream a series of objects to be serialized, the ASP.NET Core infrastructure handles the details of this for you. You can return an `IEnumerable` or `IAsyncEnumerable` from a controller and it will stream data from it automatically until the data runs out or the client disconnects.

There may be circumstances in which you want to control the enumeration process more finely, such as linking certain operations in an `IAsyncEnumerable` to other `CancellationToken`s to enforce timeouts. In this case, you can return an `IAsyncEnumerable` implementation that yields the desired sequence. It is important to include the ASP.NET Core stack's `RequestAborted` cancellation token in your operations, so that if the client disconnects mid-way, this translates to an immediate cancellation of the underlying operation. If you need to provide your own reasons for cancellation in addition, such as timeouts, you can use the method `CancellationTokenSource.CreateLinkedTokenSource` to derive a token that will signal cancellation as soon as any of the linked tokens does so.

## Structured Data with Alternative Serializers

If your web project is not using System.Text.Json to serialize data, then `IAsyncEnumerable` may not be fully supported. For serializers with no async serialization support, ASP.NET Core will attempt to automatically buffer the entire enumeration before sending it to the serializer as a conventional collection. This approach can work when the length of the enumeration is fixed and relatively small, but the following circumstances are not compatible with this workaround:

* You are delivering a very large amount of data.
* You are delivering data on-demand, and it is important that elements be sent as soon as they are available.
* Your enumeration does not have an end of its own, and continues as long as the client accepts data.

In these cases, if you return an `IAsyncEnumerable` from your controller (including one implemented implicitly within a controller method using `yield return`), your service will attempt to buffer all of the data, and will encounter an error such as:

> System.InvalidOperationException: 'AsyncEnumerableReader' reached the configured maximum size of the buffer when enumerating a value of type 'TestController+<Get>d__4'. This limit is in place to prevent infinite streams of 'IAsyncEnumerable<>' from continuing indefinitely. If this is not a programming mistake, consider ways to reduce the collection size, or consider manually converting 'TestController+<Get>d__4' into a list rather than increasing the limit.

In this case, you will need to explicitly implement the enumeration yourself. If you are using a serializer that does not have async serialization support, then any object being sent will need to be fully serialized before it can be sent to the client. However, by taking charge of the JSON array structure, you can turn each element into an independent object for serialization. The enumeration as a whole does not need to be serialized before data can be sent.
	
### Controllers

Typical controller implementations separate handling the request from delivering the result to the client. To support situations where the implementation needs tighter coupling to the process of writing results to the client, ASP.NET Core provides the type `IActionResult`, which allows you to defer actually producing the result from the controller. At a point in time after your controller's route method has returned, ASP.NET Core will call into the `ExecuteAsync` method of the class implementing the interface when results actually need to be sent to the client, giving over full control over how the data is sent.

```
public class AsyncEnumerableActionResult<T> : IActionResult
{
  IAsyncEnumerable<T> _dataSource;
  
  public MyActionResult(IAsyncEnumerable<T> dataSource)
  {
    _dataSource = dataSource;
  }
  
  static readonly byte[] JSONArrayStart = new byte[] { (byte)'[' };
  static readonly byte[] JSONArraySeparator = new byte[] { (byte)',' };
  static readonly byte[] JSONArrayEnd = new byte[] { (byte)']' };

  public async Task ExecuteResultAsync(ActionContext context)
  {
    await context.HttpContext.Response.Write(JSONArrayStart, context.HttpContext.RequestAborted);

    bool isFirstItem = true;
  
    await foreach (T item in _dataSource.WithCancellation(context.HttpContext.RequestAborted))
    {
      if (!isFirstItem)
        await context.HttpContext.Response.Write(JSONArraySeparator, context.HttpContext.RequestAborted);
  
      var itemJson = JsonConvert.SerializeObject(item);
  
      await context.HttpContext.Response.WriteAsync(itemJson, context.HttpContext.RequestAborted);
  
      isFirstItem = false;
    }
  
    await context.HttpContext.Response.Write(JSONArrayEnd, context.HttpContext.RequestAborted);
  }
}

public class MyController
{
  IDataSourceFactory _dataSourceFactory;
  
  public MyController(IDataSourceFactory dataSourceFactory)
  {
    _dataSourceFactory = dataSourceFactory;
  }

  [HttpGet]
  public IActionResult GetData()
  {
    return new AsyncEnumerableActionResult(_dataSourceFactory.GetDataSource());
  }
}
```

## Middleware

Sometimes, it is necessary to stream data from ASP.NET Core Middleware, handling requests without them being matched up with handlers in controllers by endpoint routing. In this situation, the middleware class is directly handed an `HttpContext` object to work with. It is possible, then, to directly write to the `Body` stream within the middleware's `InvokeAsync` method.

```
public class MyMiddleware
{
  RequestDelegate _next;
  IDataSourceFactory _dataSourceFactory;

  public MyMiddleware(RequestDelegate next, IDataSourceFactory dataSourceFactory)
  {
    _next = next;
    _dataSourceFactory = dataSourceFactory;
  }
  
  bool ShouldProcessRequest(HttpContext context)
  {
    // TODO: supply implementation as needed for this middleware
  }
  
  public Task InvokeAsync(HttpContext context)
  {
    if (!ShouldProcessRequest(context))
      return _next(context);
    else
      return InvokeAsyncImplementation(context);
  }
  
  static readonly byte[] JSONArrayStart = new byte[] { (byte)'[' };
  static readonly byte[] JSONArraySeparator = new byte[] { (byte)',' };
  static readonly byte[] JSONArrayEnd = new byte[] { (byte)']' };

  async Task InvokeAsyncImplementation(HttpContext context)
  {
    await context.HttpContext.Response.Write(JSONArrayStart, context.HttpContext.RequestAborted);

    bool isFirstItem = true;
  
    await foreach (T item in _dataSource.WithCancellation(context.HttpContext.RequestAborted))
    {
      if (!isFirstItem)
        await context.HttpContext.Response.Write(JSONArraySeparator, context.HttpContext.RequestAborted);
  
      var itemJson = JsonConvert.SerializeObject(item);
  
      await context.HttpContext.Response.WriteAsync(itemJson, context.HttpContext.RequestAborted);
  
      isFirstItem = false;
    }
  
    await context.HttpContext.Response.Write(JSONArrayEnd, context.HttpContext.RequestAborted);
  }
}
```

## Raw Data

Sometimes the data returned by a service is not a structured object that be returned e.g. as JSON, but a raw byte stream. For instance, this may be from the output of a video codec or the creation of a file archive (provided that the format supports being written from start to finish without seeking).

When writing raw data to the underlying stream, it will be necessary for this data to be provided in chunks that can be sent to the client in a loop. For instance, if the source of the data is another `Stream` object from which the data to be sent to the client is read, the loop might look like this:

```
HttpContext httpContext = ...;
Stream sourceStream = ...;

byte[] buffer = new byte[8192];

while (true)
{
  int numBytesRead = await sourceStream.ReadAsync(buffer, 0, buffer.Length, httpContext.RequestAborted);
  
  if (numBytesRead <= 0)
    break;
    
  await httpContext.Body.WriteAsync(buffer, 0, numBytesRead, httpContext.RequestAborted);
}
```

(This code would be placed within the `ExecuteResultAsync` method of an `IActionResult` implementation or the `InvokeAsync` method of a middleware class.)

## Buffering

Generally speaking, buffering greatly improves the performance of services performing I/O. ASP.NET Core will automatically perform buffering in most situations. This may affect the ability of your service to deliver data in real time.

The underlying HTTP response stream is buffered by the server hosting your ASP.NET Core web application. This means that data sent to the client will not actually be delivered until it has reached a critical threshold in size. If you are sending small objects that need to arrive in a timely manner, such as a stream of indeterminate length of notification packets, this buffering can introduce delays into the delivery of the messages.

If you supply an `IAsyncEnumerable` to the ASP.NET Core web stack, it will disable this buffering, and elements of the enumeration are delivered to the client as soon as they become available. If your source of items is a synchronous `IEnumerable`, then a considerable amount of data may be buffered before anything is sent to the client.

If your data needs to be delivered in a timely manner, such that records produced up to now need to be delivered now and not wait for further data to be produced, then consider using an `IAsyncEnumerable` to yield the items. If you must use an `IEnumerable`, then you will need to explicitly flush the response stream after each item. This will require you to take charge of the JSON array structure. This example shows how you might do this within an `IActionResult` implementation:

```
  static readonly byte[] JSONArrayStart = new byte[] { (byte)'[' };
  static readonly byte[] JSONArraySeparator = new byte[] { (byte)',' };
  static readonly byte[] JSONArrayEnd = new byte[] { (byte)']' };

  public async Task ExecuteResultAsync(ActionContext context)
  {
    await context.HttpContext.Response.Write(JSONArrayStart, context.HttpContext.RequestAborted);

    bool isFirstItem = true;
  
    await foreach (T item in _dataSource.WithCancellation(context.HttpContext.RequestAborted))
    {
      if (!isFirstItem)
        await context.HttpContext.Response.Write(JSONArraySeparator, context.HttpContext.RequestAborted);
  
      await context.HttpContext.Response.WriteAsync(item, context.HttpContext.RequestAborted);
      await context.HttpContext.Response.Body.FlushAsync();
  
      isFirstItem = false;
    }
  
    await context.HttpContext.Response.Write(JSONArrayEnd, context.HttpContext.RequestAborted);
  }
```

## Aborted Requests

In order to reduce the number of exceptions and other log alerts sent to diagnostics, the internal implementation of ASP.NET Core intentionally ignores write errors during the handling of a request. This situation would arise for instance if the client disconnects from the server while the request is still being delivered, which could result from a browser tab being closed.

For typical requests, this is a sound strategy, because the quantity of data being delivered is low and the complexity avoided by having to abort the request processing early means the code will be simpler and more reliable.

For requests that deliver large amounts of data, it can introduce inefficiency, because following the client's disconnection, the remaining data must still be collected and delivered to the response stream, only for it to be ignored.

For requests that deliver an indeterminate amount of data, this is a problem because the data producer will never be notified that it no longer has a consumer. For each request of this type, a server that naively continues to write data indefinitely will produce a task that runs forever within the web server. Each such task slows down the server and consumes resources, which could eventually lead to the server no longer functioning properly. External resources can also be affected if they are implicated in the processing of the request.

In order to account for this, ASP.NET Core code that is streaming responses to clients must explicitly involve `HttpContext.RequestAborted` (a `CancellationToken`) in its operations.

* Async operations reading data from the source upstream should include this as their cancellation token.
* Synchronous operations reading data from upstream should explicitly check `IsCancellationRequested` after each call returns.
* All calls to send data to the response's body stream should be made using `Async` methods, and should supply `HttpContext.RequestAborted` as `CancellationToken`.

The natural assumption that the body stream will detect that it can no longer deliver data to the client and raise an exception is incorrect.
	
### Timeouts and Other Sources of Cancellation
	
Asynchronous calls cannot be given multiple `CancellationToken`s simultaneously. As such, you cannot provide your own `CancellationToken` at the same time as `HttpContext.RequestAborted`. There is a simple solution to this problem, though: The `CancellationTokenSource.CreateLinkedTokenSource` method, which has several overloads, allows you to derive a single `CancellationToken` that will signal cancellation as soon as any of the supplied `CancellationToken`s enters the cancelled state.
	
Example:

```
public async Task StreamAsyncEnumerableWithTimeout(Stream stream, IAsyncEnumerable<T> dataSource, TimeSpan timeout, CancellationToken externalCancellationToken)
{
  using (var cancellationTokenSource = CancellationTokenSource.CreateLinkedTokenSource(HttpContext.RequestAborted, externalCancellationToken))
  {
    cancellationTokenSource.CancelAfter(timeout);
  
    var linkedToken = cancellationTokenSource.Token;

    await stream.WriteAsync(JSONArrayStart, linkedToken);

    bool first = true;

    foreach (var item in dataSource)
    {
      if (first)
        first = false;
      else
        await stream.WriteAsync(JSONArraySeparator, linkedToken);
					
      await JsonSerializer.SerializeAsync(stream, item, serializerOptions, linkedToken);
    }

    await stream.WriteAsync(JSONArrayEnd);
  }
}
```
	
It is important to ensure, as shown, that the `CancellationTokenSource` is `Dispose`d (such as with a `using` block), otherwise your service will leak resources.

## Complete Examples

The following examples, written both as a controller using `IActionResult` and as middleware, take song lyrics and repeat them indefinitely until the caller disconnects.
  
The `iAsyncEnumerable` example assumes that System.Text.Json is being used for JSON serialization and shows the simple path that leverages functionality built into ASP.NET Core.
  
The `IEnumerable` example assumes that Newtonsoft.Json is being used for JSON serialization and shows how the JSON array can be serialized in real time, as items arrive, without needing the entire enumeration to be read before the response can be started.

### Source Code

A solution containing all of the examples presented here is available in the following GitHub repository:

https://github.com/logiclrd/ASPNetCoreStreamingExample

### `IAsyncEnumerable` with System.Text.Json

```
  public interface ILyricsSource
  {
    IAsyncEnumerable<string> GetSongLyricsAsync(CancellationToken token);
  }

  [Route("/v1")]
  public class SongLyricsController : Controller
  {
    ILyricsSource _lyricsSource;

    public SongLyricsController(ILyricsSource lyricsSource)
    {
      _lyricsSource = lyricsSource;
    }

    [HttpGet("sing")]
    public IActionResult PerformSong()
    {
      return _lyricsSource.GetSongLyricsAsync(HttpContext.RequestAborted);
    }
  }
```

### `IEnumerable` with Newtonsoft.Json

#### Controller Example

```
  public interface ILyricsSource
  {
    IEnumerable<string> GetSongLyrics();
  }

  public class SongLyricsResult : IActionResult
  {
    ILyricsSource _lyricsSource;

    public SongLyricsResult(ILyricsSource lyricsSource)
    {
      _lyricsSource = lyricsSource;
    }

    static readonly byte[] JSONArrayStart = new byte[] { (byte)'[' };
    static readonly byte[] JSONArraySeparator = new byte[] { (byte)',' };

    public async Task ExecuteResultAsync(ActionContext context)
    {
      await context.HttpContext.Response.Body.WriteAsync(JSONArrayStart, context.HttpContext.RequestAborted);
  
      while (true)
      {
        foreach (string line in _lyricsSource.GetSongLyrics())
        {
          await JsonSerializer.SerializeAsync(context.HttpContext.Response.Body, line, cancellationToken: context.HttpContext.RequestAborted);
          await context.HttpContext.Response.Body.WriteAsync(JSONArraySeparator, context.HttpContext.RequestAborted);

          await context.HttpContext.Response.Body.FlushAsync(context.HttpContext.RequestAborted);

          await Task.Delay(200);
        }
      }

      // No JSONArrayEnd needs to be written, because the above loop will never exit.
    }
  }

  [Route("/v1")]
  public class SongLyricsController : Controller
  {
    ILyricsSource _lyricsSource;

    public SongLyricsController(ILyricsSource lyricsSource)
    {
      _lyricsSource = lyricsSource;
    }

    [HttpGet("sing")]
    public IActionResult PerformSong()
    {
      return new SongLyricsResult(_lyricsSource);
    }
  }
```

#### Middleware Example

```
  public interface ILyricsSource
  {
    IEnumerable<string> GetSongLyrics();
  }

  public static class SongLyricsMiddlewareExtensions
  {
    public static IApplicationBuilder UseSongLyrics(this IApplicationBuilder builder)
      => builder.UseMiddleware<SongLyricsMiddleware>();
  }

  public class SongLyricsMiddleware
  {
    RequestDelegate _next;
    ILyricsSource _lyricsSource;

    public SongLyricsMiddleware(RequestDelegate next, ILyricsSource lyricsSource)
    {
      _next = next;
      _lyricsSource = lyricsSource;
    }

    public Task InvokeAsync(HttpContext context)
    {
      if (!context.Request.Path.StartsWithSegments(new PathString("/middleware/sing")))
        return _next(context);
      else
        return InvokeAsyncImplementation(context);
    }

    static readonly byte[] JSONArrayStart = new byte[] { (byte)'[' };
    static readonly byte[] JSONArraySeparator = new byte[] { (byte)',' };

    public async Task InvokeAsyncImplementation(HttpContext context)
    {
      await context.Response.Body.WriteAsync(JSONArrayStart, context.RequestAborted);

      while (true)
      {
        foreach (string line in _lyricsSource.GetSongLyrics())
        {
          await JsonSerializer.SerializeAsync(context.Response.Body, line, cancellationToken: context.RequestAborted);
          await context.Response.Body.WriteAsync(JSONArraySeparator, context.RequestAborted);

          await context.Response.Body.FlushAsync(context.RequestAborted);

          await Task.Delay(200);
        }
      }

      // No JSONArrayEnd needs to be written, because the above loop will never exit.
    }
  }
  
  public class Startup
  {
    ...
    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
      ...

      // Register our middleware.
      app.UseSongLyrics();

      ...
    }
  }
```
