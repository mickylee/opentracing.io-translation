# 常见用例

本章的主要目的是，针对通过使用OpenTracing API来监控应用程序或类库的开发者，提供示例说明。

## 回到伊始：OpenTracing是为了哪些人建立的？

OpenTracing是一个轻量级的标准化层，它位于应用程序/类库 和 追踪或日志分析程序 之间。

~~~
   +-------------+  +---------+  +----------+  +------------+
   | Application |  | Library |  |   OSS    |  |  RPC/IPC   |
   |    Code     |  |  Code   |  | Services |  | Frameworks |
   +-------------+  +---------+  +----------+  +------------+
          |              |             |             |
          |              |             |             |
          v              v             v             v
     +-----------------------------------------------------+
     | · · · · · · · · · · OpenTracing · · · · · · · · · · |
     +-----------------------------------------------------+
       |               |                |               |
       |               |                |               |
       v               v                v               v
 +-----------+  +-------------+  +-------------+  +-----------+
 |  Tracing  |  |   Logging   |  |   Metrics   |  |  Tracing  |
 | System A  |  | Framework B |  | Framework C |  | System D  |
 +-----------+  +-------------+  +-------------+  +-----------+
~~~

**Application Code, 应用程序代码**： 开发者在开发业务代码时，可以通过OpenTracing来描述追踪数据间的因果关系，控制流程，增加细粒度的日志信息。

**Library Code, 类库代码**：类似的，类库程序作为请求控制的中介媒介，也可以通过OpenTracing来描述追踪数据间的因果关系，控制流程，增加细粒度的日志信息。例如：一个web中间件类库，可以使用OpenTracing，在请求被处理时新增span；或者，一个ORM类库，可以使用OpenTracing来描述高级别的ORM语义和特定SQL查询间的关系。

**OSS Services, OSS服务（运营支持服务）**：除嵌入式类库以外，整个OSS服务可以采取OpenTracing标准来，集成分布式追踪系统来处理一个大型的分布式系统中的复杂调用关系。例如，一个HTTP的负载均衡器可以使用OpenTracing标准来设置请求（如：设置请求图），或者一个基于键值对的存储系统使用OpenTracing来解读系统的读写性能。

**RPC/IPC Frameworks，RPC/IPC框架（远程调用框架）**：任何一个跨进程的子任务，都可以通过使用OpenTracing，来标准化追踪数据注入到传输协议中的格式。

所有上面这些，都应该使用OpenTracing来描述和传递分布式追踪数据，**而不需要了解OpenTracing的实现**。

### OpenTracing 优先级

由于OpenTracing层的 *上层* 有更多的应用程序和开发者（而不是下层），API和用例的易用性也倾向于他们。这篇文档中的用例将面向OpenTracing API调用者（而非被调者），帮助他们在建立辅助的类库和各种抽象模型，最终有利于为OpenTracing实现者节省时间和精力。

让我们直接进入主题：

## 用例

### 追踪Function（函数）

```python
def top_level_function():
    span1 = tracer.start_span('top_level_function')
    try:
        . . . # business logic，业务逻辑
    finally:
        span1.finish()
```

后续，作为业务逻辑的一部分，我们调用了`function2`方法，也想被追踪。为了让这个追踪附着在正在进行的追踪上（和上述的追踪形成一根调用链）。我们将在后面的t章节讨论如何实现，现在，我们假设一个`get_current_span`函数可以完成这个功能：

```python
def function2():
    span2 = get_current_span().start_child('function2') \
        if get_current_span() else None
    try:
        . . . # business logic
    finally:
        if span2:
            span2.finish()
```

我们假设，如果这个追踪还未被启动，无论什么原因，开发者都不想在这个函数内启动一个新的追踪，所以我们考虑到`get_current_span`函数可能返回`None`。

这两个例子都非常的简单。 通常情况下，应用程序不希望追踪代码和业务代码混在一起，而使用其他方式，例如：标注等，参考[function decorator in Python](https://github.com/uber-common/opentracing-python-instrumentation/blob/master/opentracing_instrumentation/local_span.py#L59):

```python
@traced_function
def top_level_function():
    ... # business logic
```

### 服务端追踪

当一个应用服务器要追踪一个请求的执行情况，他一般需要以下几步：

  1. 试图从请求中获取传输过来的SpanContext（防止调用链在客户端已经开启），如果无法获取SpanContext，则新开启一个追踪。
  1. 在_request context_中存储最新d创建的span，_request context_会通过应用程序代码或者RPC框架进行传输
  1. 最终，当服务端完成请求处理后，使用 `span.finish()`关闭span。

#### 从请求中获取（Extracting）SpanContext

假设，我们有一个HTTP服务器，SpanContext通过HTTP头从客户端传递到服务端，可通过`request.headers`访问到：

```python
extracted_context = tracer.extract(
    format=opentracing.HTTP_HEADER_FORMAT,
    carrier=request.headers
)
```
 这里，我们使用`headers`中的map作为carrier。追踪程序知道需要hearder的哪些内容，用来重新构建tracer的状态和Baggage。

#### 从请求中获取一个已经存在的追踪，或者开启一个新的追踪

如果无法在请求的相关的头信息中获取所需的值，上文中的`extracted_context`可能为`None`：此时我们假设客户端没有发送他们。在这种情况下，服务端需要新创建一个追踪（新调用链）。

```python
extracted_context = tracer.extract(
    format=opentracing.HTTP_HEADER_FORMAT,
    carrier=request.headers
)
if extracted_context is None:
    span = tracer.start_span(operation_name=operation)
else:
    span = tracer.start_span(operation_name=operation, child_of=extracted_context)
span.set_tag('http.method', request.method)
span.set_tag('http.url', request.full_url)
```

可以通过调用`set_tag`，在Span中记录请求的附加信息。

上面提到的`operation`是通过提供的服务名指定Span的名称。例如,如果HTTP请求到`/save_user/123`，那么`operation`名称应该被设置为`post:/save_user/`。OpenTracing API不会强制要求应用程序如何给span命名。

#### 进程内请求上下文传输

请求的上下文传输是指，对于一个请求，所有处理这个请求的层都需要可以访问到同一个_context_（上下文）。可以通过特定值，例如：用户id、token、请求的截止时间等，获取到这个_context_（上下文）。也可以通过这种方法获取正在追踪的Span。

请求_context_（上下文）的传输不属于OpenTracing API的范围，但是，这里提到他，是为了让大家更好的理解后面的章节。下面有两种常用的上下文传输技术：

##### 隐式传输

隐式传输技术要求_context_（上下文）需要被存储到平台特定的位置，允许从应用程序的任何地方获取这个值。常用的RPC框架会利用thread-local 或 continuation-local存储机制，或者全局变量（如果是单线程处理）。

这种方式的缺点在于，有明显的性能损耗，有些平台比如Go不知道基于thread-local的存储，隐式传输将几乎不可能实现。

##### 显示传输

显示传输技术要求应用程序代码，包装并传递_context_（上下文）对象：

```go
func HandleHttp(w http.ResponseWriter, req *http.Request) {
    ctx := context.Background()
    ...
    BusinessFunction1(ctx, arg1, ...)
}

func BusinessFunction1(ctx context.Context, arg1...) {
    ...
    BusinessFunction2(ctx, arg1, ...)
}

func BusinessFunction2(ctx context.Context, arg1...) {
    parentSpan := opentracing.SpanFromContext(ctx)
    childSpan := opentracing.StartSpan(
        "...", opentracing.ChildOf(parentSpan.Context()), ...)
    ...
}
```

显示传输的缺点在于，它向应用程序代码，暴露了底层的实现。[Go blog post](https://blog.golang.org/context)这边文章提供了这种方式的深层次的解析。

### 追踪客户端调用

当一个应用程序作为一个RPC客户端时，它可能希望在发起调用之前，启动一个新的追踪的span，并将这个心的span随请求一起传输。下面，通过一个HTTP请求的实例，展现如何做到这点。

```python
def traced_request(request, operation, http_client):
    # retrieve current span from propagated request context
    parent_span = get_current_span()

    # start a new span to represent the RPC
    span = tracer.start_span(
        operation_name=operation,
        child_of=parent_span.context,
        tags={'http.url': request.full_url}
    )

    # propagate the Span via HTTP request headers
    tracer.inject(
        span.context,
        format=opentracing.HTTP_HEADER_FORMAT,
        carrier=request.headers)

    # define a callback where we can finish the span
    def on_done(future):
        if future.exception():
            span.log(event='rpc exception', payload=exception)
        span.set_tag('http.status_code', future.result().status_code)
        span.finish()

    try:
        future = http_client.execute(request)
        future.add_done_callback(on_done)
        return future
    except Exception e:
        span.log(event='general exception', payload=e)
        span.finish()
        raise
```

  * `get_current_span()`函数不是OpenTracing API的一部分。它仅仅代表一个工具类的方法，通过当前的请求上下文获取当前的span。（在Python一般会这样用）。
  * 我们假定HTTP请求是异步的，所以他会返回一个Future。我们为这次调用增加的成功回调函数，在回调函数内部完成当前的span。
  * 如果HTTP客户端返回一个异常，则通过log方法将异常记录到span中。
  * 因为HTTP请求可以在返回Future后发生异常，我们使用try/catch块，在任何情况下都会完成span，保证这个span会被上报，并避免内存溢出。


### 使用 Baggage / 分布式上下文传输

上面通过网络在客户端和服务端间传输的Span和Trace，包含了任意的Baggage。客户端可以使用Baggage将一些额外的数据传递到服务端，以及这个服务端的下游其他服务器。

```python
# client side
span.context.set_baggage_item('auth-token', '.....')

# server side (one or more levels down from the client)
token = span.context.get_baggage_item('auth-token')
```

### Logging事件

我们在客户端span的示例代码中，已经使用过`log`。事件被记录不会有额外的负载，也不一定必须在span创建或完成时进行操作。例如，应用通过可以在执行过程中，通过获取当前请求的当前span,记录一个缓存未命中事件：

```python
span = get_current_span()
span.log(event='cache-miss')
```

tracer会为事件自动增加一个时间戳，这点和Span的tag操作时不同的。也可以将外部的时间戳和事件相关联，例如，[Log (Go)](https://github.com/opentracing/opentracing-go/blob/ca5c92cf/span.go#L53)。

### 使用外部的时间戳，记录Span

因为多种多样的原因，有些场景下，会将OpenTracing兼容的tracer集成到一个服务中。例如,一个用户有一个日志文件，其中包含大量的来自黑盒进程（如：HAProxy）产生的span。为了让这些数据接入OpenTracing兼容的系统，API需要提供一种方法通过外部的时间戳记录span的信息。

```python
explicit_span = tracer.start_span(
    operation_name=external_format.operation,
    start_time=external_format.start,
    tags=external_format.tags
)
explicit_span.finish(
    finish_time=external_format.finish,
    bulk_logs=map(..., external_format.logs)
)
```

### 在追踪开始之前，设置采样优先级

很多分布式追踪系统，通过采样来降低追踪数据的数量。有时，开发者想有一种方式，确保这条trace一定会被记录（采样），例如：HTTP请求中包含特定的参数，如`debug=true`。OpenTracing API标准化了一些有用的tag，其中一个被叫做"sampling priority"（采样优先级）：精确的语义是由追踪系统的实现者决定的，但是任何值大于0（默认）代表一条trace的高优先级。为了将`debug`属性传递给追踪系统，需要在追踪前进行预处理，如下面所写的这样：

```python
if request.get('debug'):
    span = tracer.start_span(
        operation_name=operation,
        tags={tags.SAMPLING_PRIORITY: 1}
    )
```
