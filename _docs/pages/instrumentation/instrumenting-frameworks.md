# 监控框架

*追踪所有的事情!*

## 谁应该阅读本章节？

这篇指导文档，面向希望将[OpenTracing](http://opentracing.io/)将入到web、RPC或其他框架的监控当中的开发者。增加监控能力可以使框架能够和端到端的分布式追踪进行整合。

当一个请求跨越一套分布式系统时，分布式追踪可以提供请求在分布式系统内的运行情况。OpenTracing是一个开源的API标准，致力于分布式请求的追踪，保证能够追踪用户从web或移动端到后台应用，以及最终的数据存储。一旦OpenTracing完成跨应用栈（跨进程）的整合，在一个分布式系统中进行追踪将更容易。这将可以满足，开发者和运维人员对于产品服务优化和加强健壮性的要求。

在开始之前，请确保有你的平台（编程语言）有对应的OpenTracing API的实现。查看[这里](/pages/api/api-implementations)。

## 总览

总体来说，集成OpenTracing，你需要做下面两件事：

服务端框架修改需求：

* 过滤器、拦截器、中间件或其他处理输入请求的组件
* span的存储，存储一个request context或者request到span的映射表
* 通过某种方式对tracer进行配置

客户端框架修改需求：

* 过滤器、拦截器、中间件或其他处理对外调用的请求的组件
* 通过某种方式对tracer进行配置

## 重要提醒：

在我们专注于实现之前，有几个重要的概念和特性需要框架开发者所熟悉。

### Operation Names，操作名

你会注意到operation_name（操作名）这个变量出现在这篇文章的各处。每一个span都需要通过一个operation_name创建，operation_name需要遵守规范的要求，[点击查看](/pages/spec)。每一个span都需要一个默认的operation_name，并提供一种可以由用户命名的方式。

默认operation_name示例：

* request handler的方法名
* web请求路径
* RPC的服务名+方法名

### 确定需要追踪的请求

有些用户希望追踪所有的请求，同时，有些用户只需要追踪特定的请求。你应该允许用户去设置是否需要追踪，以满足这两种场景。例如，你可以提供@Trace标注，被标注的方法会被追踪。你也可以提供一种配置，允许用户去设置他们是否使用标准，所有的请求是不是应该被追踪。

### 追踪请求的属性

用户可能需要追踪关于请求的一些信息，而不希望去操作span或者为span设置tag。为用户提供一种方式设置需要追踪的请求的属性，并自动追踪这些属性值，是十分有帮助的。概念上，这和gRPC中的span的Decorator函数十分类似：

```
// SpanDecorator binds a function that decorates gRPC Spans.
func SpanDecorator(decorator SpanDecoratorFunc) Option {
	return func(o *options) {
		o.decorator = decorator
	}
}
```
另一种方式，是设置`TRACED_REQUEST_ATTRIBUTES`，允许用户传递一个列表（例如：`URL`, `METHOD`, `HEADERS`），然后你会在追踪过滤器中，包含这些属性：
```
for attr in settings.TRACED_REQUEST_ATTRIBUTES:
    if hasattr(request, attr):
        payload = str(getattr(request, attr))
        span.set_tag(attr, payload)
```

## 服务端追踪

服务端追踪的目的是追踪请求在这个服务器内部的全生命周期的情况，并保证能够和前置的客户端追踪信息连接起来。你可以在服务器收到请求时，创建span，并在服务器完成请求处理后，关闭这些span。追踪一个服务端请求的流程如下：


* 服务器接收到请求
    * 从网络请求（跨进程的调用：HTTP等）获取当前的追踪链状态
    * 创建一个新的span
    * 保存当前的追踪状态
* 服务器完成请求处理 / 返回响应
    * 结束上面创建的span

由于调用流程决定于请求的处理情况，所以你需要知道如果修改框架的请求和响应处理——是否需要通过修改过滤器、中间件、配置栈或者其他机制。

### 获取当前的追踪链状态

为了在分布式系统中，跨进程边界追踪调用情况，RPC服务需要能够衔接每一个服务请求的服务端和客户端。OpenTracing允许通过inject和extract方法，将span的上下文信息编码到carrier中。（编码规范留给开发者确定，所以你需要担心这个问题。）

如果客户端发起一个请求时，span的上下文就已经被加到了请求内容中。你的工作是使用io.opentracing.Tracer.extract方法，从请求中获取span的上下文。carrier通过你使用哪种服务，决定是否哪种方法从请求中获取上下文；例如，web服务通过HTTP头作为carrier，从HTTP请求中获取span上下文（如下所示）：

Python:

```Python
span_ctx = tracer.extract(opentracing.Format.HTTP_HEADERS, request.headers)
```

Java:

```Java
import io.opentracing.propagation.Format;
import io.opentracing.propagation.TextMap;

Map<String, String> headers = request.getHeaders();
SpanContext parentSpan = tracer.getTracer().extract(Format.Builtin.HTTP_HEADERS,
    new TextMapExtractAdapter(headers));
```

OpenTracing当提取失败时，可以选择抛出异常，所以确保会捕获异常，防止异常造成服务器宕机。这种情况通常意味着请求来自于第三方应用（没有被追踪的应用），此时应该开启一个新的追踪。

### 启动一个span

一旦你接收到一个请求，并且获取到了span的上下文，你应该立即为这次请求创建一个span，代表这次请求的全生命周期。如果存在被提取出来的上下文，则新的服务端server应该是被提取出的span的孩子节点（ChildOf关系），代表客户端和服务端之间的调用关系。如果没有被注入的span，你需要启动一个新的span（没有上下级关系）。

Python:

```Python
if(extracted_span_ctx):
    span = tracer.start_span(operation_name=operation_name,
        child_of=extracted_span_ctx)
else:
    span = tracer.start_span(operation_name=operation_name)
```

Java:

```Java
if(parentSpan == null){
    span = tracer.buildSpan(operationName).start();
} else {
    span = tracer.buildSpan(operationName).asChildOf(parentSpan).start();
}
```

### 保存当前的span上下文

在处理请求期间，让用户可以访问span上下文是十分重要的。只有获取上下文，才能为服务端，进行自定义的tag设置，记录事件(log event)，创建子级的span，用于最终展现服务内部的工作情况。为了满足这个目标，你必须决定如何让用户访问当前的span。这将由框架的架构决定。这里有两个常见用例：

1. 使用请求上下文：如果你的框架有一个请求上下文，上下文可以存储任意值，这样你可以在请求处理过程中，一直把现在的span存储到上下文中。如果你的框架中有过滤器（Filter），这种实现方式是一种很好的方式。例如你有一个请求上下文叫做ctx，那么你可以这样实现一个过滤器（Filter）：

```
def filter(request):
    span = # extract / start span from request
    with (ctx.active_span = span):
        process_request(request)
    span.finish()
```

2. 现在，在请求处理的任何时候，用户都可以通过`ctx.active_span`获取当前的span。注意，一旦请求被处理，`ctx.active_span`的值就不应该被改变。

3. 建立请求和span的映射关系：如果存在这种情况：如有可能没有一个可用的请求上下文，或者你针对请求的预处理和后处理有不同的过滤器方法， 你可以选择建立一个请求和span的映射表。其中一种实现方式是创建一个框架特有的tracer的包装器（tracer wrapper），存储这个映射表，例如：

```
class MyFrameworkTracer:
    def __init__(opentracing_tracer):
        self.internal_tracer = opentracing_tracer
        self.active_spans = {}
    def add_span(request, span):
        self.active_spans[request] = span
    def get_span(request):
        return self.active_spans[request]
    def finish_span(request):
        span = self.active_spans[request]
        span.finish()
        del self.active_spans[request]
```

4. 如果你的服务器可以并行的处理请求，请确保你的span的映射表是线程安全的。

5. 过滤器处理示例代码如下：

```
def process_request(request):
    span = # extract / start span from request
    tracer.add_span(request, span)
def process_response(request, response):
    tracer.finish_span(request)
```

6. 注意：用户在处理reponse时，调用`tracer.get_span(request)`获取当前的span，请确保用户依然能获取request实例。（也可以不使用request对象，而使用其他可以标识当前请求的参数）

## 客户端追踪

当框架有一个客户端组件的时候，需要在初始化request的时候，开启客户端的追踪。这样做是为了将生成的span放到请求头中，这样span才能请求随着请求，传递到服务端。类似于服务端追踪，你需要知道如何修改你的客户端代码，来发送请求，和接收相应。当客户端完成修改，就可以完成端到端的追踪了。

追踪一个客户端请求的流程如下：

* 准备请求对象
    * 读取现在的追踪状态
    * 新建一个span
    * 将span注入(Inject)到请求中
* 发送请求
* 接收响应
    * 完成关闭span

### 读取现在的追踪状态 / 新建一个span

正如服务端一样，我们必须知道是应该开启一个新的追踪或者和一个已有的追踪连接上。例如,一个基于微服务架构分布式架构中，一个应用可能*即是服务端又是客户端*。一个服务的提供方同时又是另一个服务的发起方，这个东西需要被联系起来。如果存在一个活跃的调用链，你需要帮他的活跃span作为父级span，并在客户端请求出开启一个新的span。否则，你需要新建没有没有父级节点的span。


How you recognize whether there is an active trace depends on how you're storing active spans. If you're using a request context, then you can do something like this:

```
if hasattr(ctx, active_span):
    parent_span = getattr(ctx, active_span)
    span = tracer.start_span(operation_name=operation_name,
        child_of=parent_span)
else:
    span = tracer.start_span(operation_name=operation_name)
```

If you're using the request-to-span mapping technique, your approach might look like:

```
parent_span = tracer.get_span(request)
span = tracer.start_span(
    operation_name=operation_name,
    child_of=parent_span)
```

You can see examples of this approach in [gRPC](https://github.com/grpc-ecosystem/grpc-opentracing/blob/master/java/src/main/java/io/opentracing/contrib/ActiveSpanSource.java) and [JDBI](https://github.com/opentracing-contrib/java-jdbi/blob/9f6259538a93f466f666700e3d4db89526eee23a/src/main/java/io/opentracing/contrib/jdbi/OpenTracingCollector.java#L153).

### Inject the Span

This is where you pass the trace information into the client's request so that the server you send it to can continue the trace. If you're sending an HTTP request, then you'll just use the HTTP headers as your carrier.

span = # start span from the current trace state
`tracer.inject(span, opentracing.Format.HTTP_HEADERS, request.headers)`

### Finish the Span

When you receive a response, you want to end the span to signify that the client request is finished. Just like on the server side, how you do this depends on how your client request/response processing happens. If your filter wraps the request directly you can just do this:

```
def filter(request, response):
    span = # start span from the current trace state
    tracer.inject(span, opentracing.Format.HTTP_HEADERS, request.headers)
    response = send_request(request)
    if response.error:
       span.set_tag(opentracing., true)
    span.finish()
```

Otherwise, if you have ways to process the request and response separately, you might extend your tracer to include a mapping of client requests to spans, and your implementation would look more like this:

```
def process_request(request):
    span = # start span from the current trace state
    tracer.inject(span. opentracing.Format.HTTP_HEADERS, request.headers)
    tracer.add_client_span(request, span)
def process_response(request, response):
    tracer.finish_client_span(request)
```

## Closing Remarks

If you'd like to highlight your project as OpenTracing-compatible, feel free to use our GitHub badge and link it to the OpenTracing website.

[![OpenTracing badge](https://github.com/opentracing/contrib/blob/master/badge/OpenTracing-enabled-blue.png)](http://opentracing.io)

`[![OpenTracing Badge](https://github.com/opentracing/contrib/blob/master/badge/OpenTracing-enabled-blue.png)](http://opentracing.io)`

Once you've packaged your implementation, email us at [community@opentracing.io](mailto://community@opentracing.io) with your implementation details (platform, description, github username) and we'll create a repo for you under [opentracing-contrib](https://github.com/opentracing-contrib/), so that others will be able to find and use your integration. You can also find there concrete examples of OpenTracing integrations into different open source projects.

If you're interested in learning more about OpenTracing, join the conversation by joining our [mailing list](https://groups.google.com/forum/#!forum/opentracing) or [Gitter](https://gitter.im/opentracing/public).
