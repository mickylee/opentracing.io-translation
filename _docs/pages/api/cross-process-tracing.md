# 每一个对Inject和Extract存在疑惑，又羞于启齿的人

开发者为应用程序增加跨进程追踪能力时，必须理解[the OpenTracing specification](/pages/spec)中定义的`Tracer.Inject(...)` 和 `Tracer.Extract(...)` 的能力。这两个方法在概念上十分强大，他允许开发人员*正确*并*抽象*的完成跨进程传输的代码，**而不需要绑定特定的OpenTracing的实现**；也就是说，强大的能力带来了巨大的困惑:)

这篇文档，针对`Inject` 和 `Extract`设计和用法，提供一个简要的总结，而不考虑特定的OpenTracingg规范各语言的实现和基于OpenTracing标准的追踪系统。

## 显示的trace传播的“重大作用”

分布式追踪系统最困难的部分就是在*分布式*的应用环境下保持追踪的正常工作。任何一个追踪系统，都需要理解多个跨进程调用间的因果关系，无论他们是通过RPC框架、发布-订阅机制、通用消息队列、HTTP请求调用、UDP传输或者其他传输模式。

一些分布式追踪系统（例如，2003年的[Project5](http://dl.acm.org/citation.cfm?id=945454)，2006年的[WAP5](http://www2006.org/programme/item.php?id=2033)，2014年的[The Mystery Machine](https://www.usenix.org/node/186168)）会*推断*跨进程间的因果关系。当然，**这些系统，都需要在基于黑盒的因果关系推断 与 追踪结果的整合、实时准确展现上，进行处理折衷。** 处于对准确展现的关注，OpenTracing是一个*明确*的分布式追踪系统标准，它更倾向于如果产品的处理方式：2007年的[X-Trace](https://www.usenix.org/conference/nsdi-07/x-trace-pervasive-network-tracing-framework)，2010年的[Dapper](http://research.google.com/pubs/pub36356.html)，以及很多开源的追踪系统，如：[Zipkin](https://github.com/openzipkin)，[Appdash](https://github.com/sourcegraph/appdash) 等等

**`Inject` 和 `Extract` 允许开发者进行跨进程追踪时，不用和特定的OpenTracing实现进行紧耦合**

## OpenTracing跨进程传播需求

为了使`Inject` 和 `Extract`有效，*必须*遵守如下要求：

- 如上文所述，[OpenTracing 用户](/pages/instrumentation/common-use-cases#stepping-back-who-is-opentracing-for) 处理跨进程的追踪传输时，必须不需要使用OpenTracing使用中的特定代码
- 基于OpenTracing的追踪系统，必须不需要针对每一种已知的跨进程通讯机制都进行处理：这其中包含太多的工作，很多还没有明确的定义
- 也就是说，这套传播机制必须是最利于扩展的

## 基本方法：Inject, Extract, 和 Carriers

追踪过程中的任何一个SpanContext可以被**Injected**（注入）到一个Carrier中。**Carrier**可以是一个接口或者一个数据载体，他对于跨进程通讯（IPC）是十分有帮助的。**Carrier**负责将追踪状态从一个进程"carries"（携带，传递）到另一个进程。OpenTracing标准包含两种[必须的 Carrier 格式](#required-carriers)，尽管，[自定义的 Carrier 格式](#custom-carriers) 也是可能的。

同样的，对于一个Carrier，如果已经被**Injected**，那么它也可以被**Extracted**（提取），从而得到一个SpanContext实例。这个SpanContext代表着被**Injected**到Carrier的信息。

#### Inject伪代码示例

```python
span_context = ...
outbound_request = ...

# 我们将使用（内建的）基于HTTP_HEADERS的carrier格式。
# 我们在调用`tracer.inject之前，先将一个空的map作为一个carrier
carrier = {}
tracer.inject(span_context, opentracing.Format.HTTP_HEADERS, carrier)

# `carrier` 现在（隐形）包含我们通过网络传输的键值对。
for key, value in carrier:
    outbound_request.headers[key] = escape(value)
```

#### Extract 伪代码示例

```python
inbound_request = ...

# 我们将再次使用基于（内建的）HTTP_HEADERS carrier格式。
# 按照HTTP_HEADERS的文档, 我们可以使用一个map来存储外来的值，
# 允许OpenTracing实现者来根据需要，
# 来查找集合内部的键值对。
#
# 也就是说，我们直接使用基于键值对的`inbound_request.headers`作为carrier。
carrier = inbound_request.headers
span_context = tracer.extract(opentracing.Format.HTTP_HEADERS, carrier)
# Continue the trace given span_context. E.g.,
span = tracer.start_span("...", child_of=span_context)

# (如果 `carrier` 保存着trace的数据， 则现在可以创建`span`了。)
```

#### Carrier格式

所有的Carrier都有自己的格式。在一些语言的OpenTracing实现中，格式必须必须作为一个常量或者字符串来指定； 另一些，则通过Carrier的静态类型来指定。

<div id="required-carriers"></div>

## Inject/Extract Carrier 所必须的格式

至少，OpenTracing标准所有平台的实现者支持两种Carrier格式：基于"text map"（基于字符串的map）的格式和基于"binary"（二进制）的格式。

- *text map* 格式的 Carrier是一个平台惯用的map格式，基于unicode编码的`字符串`对`字符串`键值对
- *binary* 格式的 Carrier 是一个不透明的二进制数组（可能更紧凑和有效）

OpenTracing的实现者选择如何将数据存储到Carrier中，OpenTracing标准没有正式定义，但是，可以推测的是，他们会通过一种方式编码“追踪状态”，来传递`SpanContext`（例如，Dapper会包含`trace_id`，`span_id`，以及一位掩码标识这个trace的采样状态）和Baggage中的其他键值对数据。

### 各种OpenTracing实现者，实现 *跨进程边界* 方式的互操作性

不能期待不同的OpenTracing实现，`Inject` 和 `Extract` SpanContexts采用相互兼容的方式。虽然OpenTracing对于实现 *跨整个分布式系统* 的追踪系统是无从得知的，为了成功实现跨进程的追踪的我收过程，跨进程追踪的两端应该使用相同的追踪系统实现。（即远程调用的两段，使用同一套tracer）。

<div id="custom-carriers"></div>

## 自定义的 Inject/Extract Carrier 格式

任何的基于网络传输的子系统（RPC库，消息队列等）可能选择引入他们自定义的Inject/Extract的Carrier格式；根据需要自定义格式，**但最终要求返回符合OpenTracing格式的结果**。这样允许OpenTracing的实现者可以优化他们自己的自定义格式，而*不需要*实现者支持这些子系统的自定义格式。

一些伪代码将可能更明确的说明这个问题。假设我们是**ArrrPC pirate RPC subsystem**的作者，我们希望增加OpenTracing的数据在RPC请求过程中传输。不考虑异常处理，我们的伪代码可能如下所示：

```python
span_context = ...
outbound_request = ...

# 首先，我们使用我们自定义的Carrier：outbound_request
# 如果我们优先支持OpenTracing的实现，这样会更加高效的处理。
# 但是，这不是一个必须的格式要求，我们不能指望基于OpenTracing的
# 追踪程序支持arrrpc.ARRRPC_OT_CARRIER参数
try:
    tracer.inject(span_context, arrrpc.ARRRPC_OT_CARRIER, outbound_request)

except opentracing.UnsupportedFormatException:
    # If unsupported, fall back on a required OpenTracing format.
    # 如果不支持，则使用OpenTracing支持的格式
    carrier = {}
    tracer.inject(span_context, opentracing.Format.HTTP_HEADERS, carrier)
    # `carrier` 现在包含键值对，我们可以使用任何网络协议，来传输这个键值对即可
    for key, value in carrier:
	outbound_request.headers[key] = escape(value)
```

<div id="format-identifiers"></div>

### 关于Carrier自定义格式的更多内容

"Carrier的格式"在不同平台可能是不一样的，但在所有的场景下，他们都会使用一个全局的命名空间。支持一个全新的自定义格式的carrier*不必*修改OpenTracing核心平台的API， 尽管每一个实现OpenTracing平台API时，必须定义符合OpenTracing标准要求的carrier格式（比如：基于字符串的map和二进制块）。例如，ArrrPC RPC的维护团队定义了一个叫做"ArrrPC"的Inject/Extract格式，他们不需要向OpenTracing团队提交PR（当然OpenTracing的实现者不要求一定支持"ArrrPC"格式）。[an end-to-end injector and extractor example below，一个端到端的injector和extractor示例](#propagation-example) 将更具体的描述这个问题。


<div id="propagation-example"></div>

## 一个端到端的injector和extractor示例

为了让描述更具体，考虑如下的流程：

1. 一个*客户端*进程有一个`SpanContext`实例，并准备进行通过自制的HTTP协议，进行一次RPC调用
1. 客户端进程调用`Tracer.Inject(...)`，传入`SpanContext`实例，支持基于字符格式的map的标识符，以及支持基于字符map的Carrier，三个参数
1. Carrier将在Carrier对基于字符的map（参数2）填充必要的数据；客户端应用程序会在自制的HTTP协议中，对这个map进行编码（如：添加到HTTP头中）
1. *进行HTTP调用，将数据进行跨进程传输*
1. 现在，在服务端进程进行处理。应用程序从自制的HTTP协议中解码出上文所述的map（参数2），并通过他，初始化基于字符map的Carrier
1. 服务端进程调用`Tracer.Extract(...)`，传入需要的操作名（operation name），支持就要字符格式的map的标识符，以及上面构建的Carrier
1. 不考虑数据丢失，和其他错误，*服务端*现在有了和客户端追踪上下文中，一样的`SpanContext`。

在[OpenTracing use cases, OpenTracing常见用例](/pages/instrumentation/common-use-cases) 文档中，可以找到其他使用案例。
