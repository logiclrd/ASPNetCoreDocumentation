# Streaming Responses in ASP.NET Core

Some web endpoints need to transfer a large amount of data to the client, such that buffering that data in memory would be impractical or impossible. In some cases, the data might be of indeterminate length, such as a live video stream, which needs to continue transmitting data as long as a client remains connected. Most aspects of ASP.NET Core are oriented toward HTTP requests that are short and complete quickly, and this is what the web protocol and server & client software is optimized for, but for some use cases, there is no getting around returning large blocks of data. It is important to design the code for such endpoints carefully, to ensure that they do not consume too many resources and that they behave properly in all cases. If problems arise, they can affect the functionality of the entire site, and, indirectly, even other resources on the same server.

The common path offered by ASP.NET Core is designed to provide the best balance of simplicity and performance for short, small requests. There are patterns that can provide effectively for delivering data without keeping it in memory all at once. Some aspects require care to ensure that they are implemented correctly.

In this topic, the term _streaming data_ is used to describe a situation where the quantity of data is either large enough that it is important to avoid buffering, or it is important to deliver content to the client immediately as it becomes available, rather than returning the entire response as a unit.

## Conventions

The sample code presented in this article assumes that you are using Dependency Injection with your ASP.NET Core service implementation. As such, when classes have parameters on the constructor and expect to be handed objects of the types they need, this is handled by means of the D.I. collection configured by the `ConfigureServices` method in the `Startup` class.

It is also assumed that types will typically be exposed by means of interfaces. This is a common practice that enables "mock" objects to be injected into instances to allow automated testing, which can greatly improve code quality. You may adapt the examples here to your project's conventions as needed.

## Controllers

If you need to stream data from an endpoint in an ASP.NET Core MVC Controller, there is a particular pattern that allows you to avoid buffering.

With a typical ASP.NET Core Controller endpoint, you simply return a model object of the type you want to be returned. When returning an indeterminate number of records that are all of the same type, it is tempting to use `IEnumerable<T>` or `IAsyncEnumerable<T>` to produce an enumeration of items to be serialized to the client. However, ASP.NET Core will try to read this enumeration through to its end before it starts writing anything to the client. If there is no end, then it will eventually run out of buffer space and fail:

> System.InvalidOperationException: 'AsyncEnumerableReader' reached the configured maximum size of the buffer when enumerating a value of type 'TestController+<Get>d__4'. This limit is in place to prevent infinite streams of 'IAsyncEnumerable<>' from continuing indefinitely. If this is not a programming mistake, consider ways to reduce the collection size, or consider manually converting 'TestController+<Get>d__4' into a list rather than increasing the limit.

The solution to this is to serialize the items to the HTTP request body stream yourself. To support this, ASP.NET Core provides the type `IActionResult`, which allows you to defer actually producing the result from the controller. At a point in time after your controller's route method has returned, ASP.NET Core will call into the `ExecuteAsync` method of the class implementing the interface when results actually need to be sent to the client, giving over full control over how the data is sent.

```
public class MyActionResult : IActionResult
{
  IDataSource _dataSource;
  
  public MyActionResult(IDataSource dataSource)
  {
    _dataSource = dataSource;
  }
  
  public async Task ExecuteResultAsync(ActionContext context)
  {
    await _dataSource.WriteToStreamAsync(context.HttpContext.Response.Body);
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
    return new MyActionResult(_dataSourceFactory.GetDataSource());
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
  
  async Task InvokeAsyncImplementation(HttpContext context)
  {
    await _dataSource.WriteToStreamAsync(context.Response.Body);
  }
}
```

## Raw Data

The preceding examples have used an abstract `IDataSource` object as the source of data to be written to the request body stream. In most applications, the exact data type will be known and will be the same for all requests. One common form of data is a raw byte stream, for instance from the output of a video codec or the creation of a file archive (provided that the format supports being written from start to finish without seeking).

When writing raw data to the underlying stream, it will be necessary for this data to be provided in chunks that can be sent to the client in a loop. For instance, if the source of the data is another `Stream` object from which the data to be sent to the client is read, the loop might look like this:

> WARNING: Incomplete code example. Do not use as-is.

```
// This code example is incomplete. Please read to the end of the article for a complete discussion.

HttpContext httpContext = ...;
Stream sourceStream = ...;

byte[] buffer = new byte[8192];

while (true)
{
  int numBytesRead = await sourceStream.ReadAsync(buffer, 0, buffer.Length);
  
  if (numBytesRead <= 0)
    break;
    
  await httpContext.Body.WriteAsync(buffer, 0, numBytesRead);
}
```

(This code would be placed within the `ExecuteResultAsync` method of an `IActionResult` implementation or the `InvokeAsync` method of a middleware class.)

## Structured Data

If the data you are returning is a series of objects produced by an `IEnumerable<T>` or `IAsyncEnumerable<T>`, then you can pass this enumerable object directly to the serializer. The serializer will automatically emit the enumerable as an array, and it will serialize each item as it is read without buffering. If the enumeration blocks while obtaining data for further records, then the stream of data to the client will simply pause until it is available. The serializer will write the end of the array only when the enumeration indicates there are no further records. If your enumeration never terminates, then the serializer will continue to write data to the client indefinitely.

> WARNING: Incomplete code example. Do not use as-is.

```
// This code example is incomplete. Please read to the end of the article for a complete discussion.

HttpContext httpContext = ...;

IEnumerable<Record> recordsToSend = ...;

await System.Text.JsonSerializer.SerializeAsync(httpContext.Response.Body, recordsToSend);
```

(This code would be placed within the `ExecuteResultAsync` method of an `IActionResult` implementation or the `InvokeAsync` method of a middleware class.)

> **Note**: As of writing (May 2020), System.Text.Json does not support serializing from `IAsyncEnumerable<T>` sequences. Only `IEnumerable<T>` can be used. This does not mean that your sequence will have to run to completion synchronously before data is sent, only that each separate item's retrieval from the enumerator will be synchronous.
>
> There is an outstanding issue for adding support for `IAsyncEnumerable<T>` to `System.Text.Json`.

## Buffering

There are some caveats and gaps with the above implementations surrounding buffering.

### HTTP Response Stream

First and foremost, the underlying HTTP response stream is buffered by the server hosting your ASP.NET Core web application. This means that data sent to the client will not actually be delivered until it has reached a critical threshold in size. If you are sending small objects that need to arrive in a timely manner, such as a stream of indeterminate length of notification packets, this buffering can introduce delays into the delivery of the messages.

If your data needs to be delivered in a timely manner, such that records produced up to now need to be delivered now and not wait for further data to be produced, then you will need to explicitly flush the response stream after you have queued up the data you need sent. This is easy to do with a single call:

> WARNING: Incomplete code example. Do not use as-is.

```
// This code example is incomplete. Please read to the end of the article for a complete discussion.
await httpContext.Body.FlushAsync();
```

### JSON Serializer

If you are using a JSON serializer to deliver structured data from an `IEnumerable<T>`, be aware also that the System.Text.Json serializer itself performs buffering of its output before delivering it to the underlying stream. This greatly improves performance. If your endpoint is producing and returning records as quickly as it can, then this buffering is a good thing and will ensure that the data is transferred to the client as efficiently as possible. However, if your data source includes blocking operations that wait for state to change, and you need the objects returned to be delivered to the client as soon as they are available, System.Text.Json's buffering will prevent this.

One way around this is to take charge of the JSON array structure, and invoke the JSON serializer separately for each item:

> WARNING: Incomplete code example. Do not use as-is.

```
// This code example is incomplete. Please read to the end of the article for a complete discussion.

static readonly byte[] JSONArrayStart = new byte[] { (byte)'[' };
static readonly byte[] JSONArraySeparator = new byte[] { (byte)',' };
static readonly byte[] JSONArrayEnd = new byte[] { (byte)']' };

public void StreamEnumerable(Stream stream, IEnumerable<T> dataSource)
{
  await stream.WriteAsync(JSONArrayStart);

  bool first = true;

  foreach (var item in dataSource)
  {
    if (first)
      first = false;
    else
      await stream.WriteAsync(JSONArraySeparator);
					
    await JsonSerializer.SerializeAsync(stream, item, serializerOptions);
  }

  await stream.WriteAsync(JSONArrayEnd);
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

## Complete Example

The following example, written both as a controller using `IActionResult` and as middleware, takes song lyrics and repeats them indefinitely until the caller disconnects.

### Source Code

A solution containing all of the examples presented here is available in the following GitHub repository:

https://github.com/logiclrd/ASPNetCoreStreamingExample

### Controller Example

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

### Middleware Example

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
