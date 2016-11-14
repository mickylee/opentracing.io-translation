# 常见用例

本章的主要目的是，针对通过使用OpenTracing API来监控应用程序或类库的开发者，提供示例说明。

## 回到开头：OpenTracing是为了哪些人建立的？

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

#### In-Process Request Context Propagation

Request context propagation refers to application's ability to associate a certain _context_ with the incoming request such that this context is accessible in all other layers of the application within the same process. It can be used to provide application layers with access to request-specific values such as the identity of the end user, authorization tokens, and the request's deadline. It can also be used for transporting the current tracing Span.

Implementation of request context propagation is outside the scope of the OpenTracing API, but it is worth mentioning them here to better understand the following sections. There are two commonly used techniques of context propagation:

##### Implicit Propagation

In implicit propagation techniques the context is stored in platform-specific storage that allows it to be retrieved from any place in the application. Often used by RPC frameworks by utilizing such mechanisms as thread-local or continuation-local storage, or even global variables (in case of single-threaded processes).

The downside of this approach is that it almost always has a performance penalty, and in platforms like Go that do not support thread-local storage implicit propagation is nearly impossible to implement.

##### Explicit Propagation

In explicit propagation techniques the application code is structured to pass around a certain _context_ object:

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

The downside of explicit context propagation is that it leaks what could be considered an infrastructure concern into the application code. This [Go blog post](https://blog.golang.org/context) provides an in-depth overview and justification of this approach.

### Tracing Client Calls

When an application acts as an RPC client, it is expected to start a new tracing Span before making an outgoing request, and propagate the new Span along with that request. The following example shows how it can be done for an HTTP request.

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

  * The `get_current_span()` function is not a part of the OpenTracing API. It is meant to represent some util method of retrieving the current Span from the current request context propagated implicitly (as is often the case in Python).
  * We assume the HTTP client is asynchronous, so it returns a Future, and we need to add an on-completion callback to be able to finish the current child Span.
  * If the HTTP client returns a future with exception, we log the exception to the Span with `log` method.
  * Because the HTTP client may throw an exception even before returning a Future, we use a try/catch block to finish the Span in all circumstances, to ensure it is reported and avoid leaking resources.


### Using Baggage / Distributed Context Propagation

The client and server examples above propagated the Span/Trace over the wire, including any Baggage. The client may use the Baggage to pass additional data to the server and any other downstream server it might call.

```python
# client side
span.context.set_baggage_item('auth-token', '.....')

# server side (one or more levels down from the client)
token = span.context.get_baggage_item('auth-token')
```

### Logging Events

We have already used `log` in the client Span use case. Events can be logged without a payload, and not just where the Span is being created / finished. For example, the application may record a cache miss event in the middle of execution, as long as it can get access to the current Span from the request context:

```python
span = get_current_span()
span.log(event='cache-miss')
```

The tracer automatically records a timestamp of the event, in contrast with tags that apply to the entire Span. It is also possible to associate an externally provided timestamp with the event, e.g. see [Log (Go)](https://github.com/opentracing/opentracing-go/blob/ca5c92cf/span.go#L53).

### Recording Spans With External Timestamps

There are scenarios when it is impractical to incorporate an OpenTracing compatible tracer into a service, for various reasons. For example, a user may have a log file of what's essentially Span data coming from a black-box process (e.g. HAProxy). In order to get the data into an OpenTracing-compatible system, the API needs a way to record spans with externally defined timestamps.

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

### Setting Sampling Priority Before the Trace Starts

Most distributed tracing systems apply sampling to reduce the amount of trace data that needs to be recorded and processed. Sometimes developers want to have a way to ensure that a particular trace is going to be recorded (sampled) by the tracing system, e.g. by including a special parameter in the HTTP request, like `debug=true`. The OpenTracing API standardizes around some useful tags, and one o them is the so-called "sampling priority": exact semnatics are implementation-specific, but any values greater than zero (the default) indicates a trace of elevated importance. In order to pass this attribute to tracing systems that rely on pre-trace sampling, the following approach can be used:

```python
if request.get('debug'):
    span = tracer.start_span(
        operation_name=operation,
        tags={tags.SAMPLING_PRIORITY: 1}
    )
```
