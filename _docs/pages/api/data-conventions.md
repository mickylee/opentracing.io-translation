# Data Conventions 数据约定

## 介绍

OpenTracing通过定义的API，可实现将监控数据记录到一个可插拔的tracer上。总体上来说，OpenTracing不能保证底层追踪系统的实现方式。那么API层应该提供什么类型的数据来保证这些底层追踪系统实现的兼容性呢？

监控软件和追踪软件开发者在高层次的共识，将产生巨大的价值：如果在一些通用的应用场景下，都使用某些已知的tag的键值对，tracer程序可以选择对他们进行特别的关注。被`log`的事件，span的结构也是如此。

例如，考虑基于HTTP的应用服务器。应用系统处理的请求中的URL、HTTP动作（get/post等）、返回码，对于应用系统的诊断是非常有帮助的。监控者可以选择使用tag方法标记这个参数，命名为`URL`或`http.url`，从纯API技术角度来说是有效的。但是，如果一个tracer需要增加一些高级功能，例如根据URL的值建立索引，或者针对特定来源的请求进行采样，你必须知道数据的格式。换句话说，tag的名字和监控程序的提供方的要求必须是一直的，这样追踪程序才能在收到数据后，提供更加智能的分析结果。

本文在这里向追踪软件的作者在纯数据收集之外，提供一个通用的指导意见。追踪系统的开发者不必严格遵守指南，但是强烈推荐大家这么做。



## Spans


### Span Naming, Span命名

Span 可以包含很多的tags、logs和baggage，但是始终需要一个高度概括的**operation name**。这些应该是一个简单的字符串，代表span中进行的工作类型。这个字符串应该是工作类型的逻辑名称，例如代表一个RPC或者一次HTTP的调用的端点，亦或对于代表SQL的span，使用`SELECT` or `INSERT`作为逻辑名，等等。

其次，span可以存在一个可选的tag，叫做**component**，他的值可以典型的代表一个进程、框架、类库或者模块名称。这个tag会很有价值。

### Span Structure, Span的结构

Span的结构也是非常重要的：span代表了什么，span和span的上下级是什么关系？这些内容在章节[Concepts and Terminology, 概念与术语](/pages/spec.html)中描述。


## Span Tag Use-Cases

The following tags are recommended for instrumentors who are trying to represent a particular type of data. Tag names follow a general structure of namespacing.

The recommended tags below are accompanied by `const` values included in an `ext` module for each opentracing implementation.  These `ext` values should be used in place of the strings below, as tracers may choose to use different underlying representations for these common concepts.  The symbols are similar in each implementation: ([Go](https://github.com/opentracing/opentracing-go/blob/master/ext/tags.go), [Python](https://github.com/opentracing/opentracing-python/blob/master/opentracing/ext/tags.py), etc.)

Some tags mentioned below may contain values of significant size. Handling of such values is implementation-specific: it is the responsibility of the Tracer to honor, drop, or truncate these tags as appropriate. However, care should be exercised on the part of the instrumentor, as even the generation or passing of such values to the Tracer may create undesirable overhead for the application.

It is not required that all suggested tags be used, even if one is used.

### Errors

The error state of a span instance should be represented as a tag.

* `error` - bool
    - `true` means that the span is in an error state
    - `false` or the absence of an `error` tag means that the span is not in an error state

### Component Identification

For any span, it can be useful to specify the type of component being instrumented. This is particularly recommended for instrumentation provided for libraries or modules, where end-users may have a mix of custom and library-provided instrumentation.

* `component` - string
    - Low-cardinality identifier of the module, library, or package that is instrumented.
    - Examples:
        - `httplib` for instrumentation of Python builtin httplib functionality
        - `JDBC` for instrumentation of JDBC database connectors
        - `mongoose` for instrumentation of Ruby MongoDB client connector
* `span.kind` - string
    - One of `client` or `server`, indicating if this span represents a client or server

### HTTP Server Tags

These tags are recommended for spans marking entry into a HTTP-based service.

* `http.url` - string
    - URL of the request being handled in this segment of the trace, in [standard URI format](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier).
    - Protocol optional
    - Examples:
        - `https://domain.net/path/to?resource=here`
        - `domain.net/path/to/resource`
        - `http://user:pass@domain.org:8888`
* `http.method` - string
    - HTTP method of the request being handled.
    - Case-insensitive
    - Examples:
        - `GET`, `POST`, `HEAD`
* `http.status_code` - integer
    - HTTP status code to be returned with HTTP response.
    - Examples:
        - `200`, `503`
* `span.kind` - string
    - Value of `server` should be used to indicate that this is a server-side span (see "<a href="#component-identification">Component Identification</a>")


### Peer Tags

These tags can be provided by either client-side or server-side to describe the downstream (client case) or upstream (server case) peer being communicated with.

* `peer.hostname` - string
    - Remote hostname
* `peer.ipv4` - string
    - Remote IP v4 address
* `peer.ipv6` - string
    - Remote IP v6 address
* `peer.port` - integer
    - Remote port
* `peer.service` - string
    - Remote service name

### Sampling

OpenTracing API does not enforce the notion of sampling, but most implementation do use it in one form or another. Sometimes the application needs to give a hint to the tracer that it would like to have a particular trace recorded in storage even if the normal sampling says otherwise. The `sampling.priority` tag is used to provide such hint. Tracing implementations are not required to respect the hint, but most will do their best to preserve the trace.

* `sampling.priority` - integer
    - If greater than 0, a hint to the tracer to do its best to capture the trace.
    - If 0, a hint to the tracer to not capture the trace.
    - If this tag is not provided, the tracer should use its default sampling mechanism.

