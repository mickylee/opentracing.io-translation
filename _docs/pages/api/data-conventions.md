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


## Span Tag Use-Cases, Span Tag操作用例

监控软件开发者，如果试图标书如下特定类型的数据，请使用下面推荐的tags。tag名称遵循命名空间的通用结构（即：java包名的结构）

下面推荐的tag，在`ext`模块中，都会为每一个实现制定一个`const`常量值。这些`ext`的值应该用来代表下面的字符串，不同的追踪系统，可以为这些通用概念选择不同的底层实现。在每种实现中，这些值的实现方式是十分相似的。(例如：[Go](https://github.com/opentracing/opentracing-go/blob/master/ext/tags.go), [Python](https://github.com/opentracing/opentracing-python/blob/master/opentracing/ext/tags.py))

下面提供的一下tags可能包含一些象征大小的值。如何处理这些值是依赖于实现的：追踪系统会需要适当选择，是否要使用、删除或者清空这些tags标记。然而，不仅仅追踪程序才需要关注这些值，给追踪系统生成、传递这些值，也可能对应用系统造成不良影响。

监控系统可以只支持其中的部分tags。

### Errors

一个span实例的错误状态，通过一个tag来标注。

* `error` - bool
    - `true` 代表这个span是错误状态
    - `false` 或没有 `error` tag ，代表span没有发生错误

### Component Identification, 框架定义

对于任何一个span，被监控的组件，指定组件的类型是十分有帮助的。十分推荐库或者模块为监控程序提供组件的定义，最终用户可能会拥有一个由框架和第三方混合提供的监控。

* `component` - string
    - 需要被检测/监控的类库、模块、包的基本名称。
    - Examples:
        - `httplib` 代表Python内建的httplib函数功能
        - `JDBC` 代表JDBC数据库连接
        - `mongoose` 代表Ruby的MongoDB客户端连接
* `span.kind` - string
    - `client` 或 `server`, 指定这个span代表一个客户端还是服务端

### HTTP Server Tags

这些tag作用于基于HTTP的服务入口的span。

* `http.url` - string
    - URL 分布式追踪中，这一阶段的调用的URL地址, 参考 [standard URI format](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier).
    - Protocol 协议，可选
    - Examples:
        - `https://domain.net/path/to?resource=here`
        - `domain.net/path/to/resource`
        - `http://user:pass@domain.org:8888`
* `http.method` - string
    - HTTP 请求被处理的方法.
    - Case-insensitive 大小写敏感
    - Examples:
        - `GET`, `POST`, `HEAD`
* `http.status_code` - integer
    - HTTP 返回值
    - Examples:
        - `200`, `503`
* `span.kind` - string
    - `server` 定义这是服务端类型的span (see "<a href="#component-identification">Component Identification, 框架定义</a>")


### Peer Tags

这些tag可以被客户端或者服务端提供，用于描述远程请求过程中，请求调用的方向。（客户端记录下行访问，服务端记录上行访问）

* `peer.hostname` - string
    - 目标 hostname
* `peer.ipv4` - string
    - 目标 IP v4 地址
* `peer.ipv6` - string
    - 目标 IP v6 地址
* `peer.port` - integer
    - 目标 port
* `peer.service` - string
    - 目标服务名称

### Sampling, 采样

OpenTracing API不强调采样的概念，但是大多数追踪系统通过不同方式实现采样。有些情况下，应用系统需要通知追踪程序，这条特定的调用需要被记录，即使根据默认采样规则，它不需要被记录。`sampling.priority` tag 提供这样的方式。追踪系统不保证一定采纳这个参数，但是会尽可能的保留这条调用。

* `sampling.priority` - integer
    - 如果大于 0, 追踪系统尽可能保存这条调用链
    - 等于 0, 追踪系统不保存这条调用链
    - 如果此tag没有提供，追踪系统使用自己的默认采样规则

