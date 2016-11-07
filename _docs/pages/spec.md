# 概念和术语


~~~
一个tracer过程中，各span的关系


        [Span A]  ←←←(the root span)
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←←(Span C 是 Span A 的孩子节点, ChildOf)
     |             |
 [Span D]      +---+-------+
               |           |
           [Span E]    [Span F] >>> [Span G] >>> [Span H]
                                       ↑
                                       ↑
                                       ↑
                         (Span G 在 Span F 后被调用, FollowsFrom)

~~~

~~~
上述tracer与span的时间轴关系


––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–––––––|–> time

 [Span A···················································]
   [Span B··············································]
      [Span D··········································]
    [Span C········································]
         [Span E·······]        [Span F··] [Span G··] [Span H··]
~~~

## Traces

一个trace代表一个潜在的，分布式的，存在并行数据或并行执行轨迹（潜在的分布式、并行）的系统。一个trace可以认为是多个span的有向无环图（DAG）。

## Spans

一个span代表系统中具有开始时间和执行时长的逻辑运行单元。span之间通过嵌套或者顺序排列建立逻辑因果关系。

### 操作名称

每一个span都有一个操作名称，这个名称简单，并具有可读性高。（例如：一个RPC方法的名称，一个函数名，或者一个大型计算过程中的子任务或阶段）。span的操作名应该是一个抽象、通用的标识，能够明确的、具有统计意义的名称；更具体的子类型的描述，请使用[Tags](#tags)

例如，假设一个获取账户信息的span会有如下可能的名称：

| 操作名 | 指导意见 |
|:---------------|:--------|
| `get` | 太抽象 |
| `get_account/792` | 太明确 |
| `get_account` | 正确的操作名，关于`account_id=792`的信息应该使用[Tag](#tags)操作 |

<span id="references"></span>

### Span间关系

一个span可以和一个或者多个span间存在因果关系。OpenTracing定义了两种关系：`ChildOf` 和 `FollowsFrom`。**这两种引用类型代表了子节点和父节点间的直接因果关系**。未来，OpenTracing将支持非因果关系的span引用关系。（例如：多个span被批量处理，span在同一个队列中，等等）

**`ChildOf` 引用:** 一个span可能是一个父级span的孩子，即"ChildOf"关系。在"ChildOf"引用关系下，父级span某种程度上取决于子span。下面这些情况会构成"ChildOf"关系：

- 一个RPC调用的服务端的span，和RPC服务客户端的span构成ChildOf关系
- 一个sql insert操作的span，和ORM的save方法的span构成ChildOf关系
- 很多span可以并行工作（或者分布式工作）都可能是一个父级的span的子项，他会合并所有子span的执行结果，并在指定期限内返回

下面都是合理的表述一个"ChildOf"关系的父子节点关系的时序图。

~~~
    [-Parent Span---------]
         [-Child Span----]

    [-Parent Span--------------]
         [-Child Span A----]
          [-Child Span B----]
        [-Child Span C----]
         [-Child Span D---------------]
         [-Child Span E----]
~~~

**`FollowsFrom` 引用:** 一些父级节点不以任何方式依然他们子节点的执行结果，这种情况下，我们说这些子span和父span之间是"FollowsFrom"的因果关系。"FollowsFrom"关系可以被分为很多不同的子类型，未来版本的OpenTracing中将正式的区分这些类型

下面都是合理的表述一个"FollowFrom"关系的父子节点关系的时序图。

~~~
    [-Parent Span-]  [-Child Span-]


    [-Parent Span--]
     [-Child Span-]


    [-Parent Span-]
                [-Child Span-]
~~~

### Logs

每个span可以进行多次**Logs**操作，每一次**Logs**操作，都需要一个带时间戳的时间名称，以及可选的任意大小的存储结构。事件名称应该是span生命周期内，一些特定事件的标识符。例如，一个代表浏览器页面加载的span，可以为每一个字段添加一个时间，可参考[Performance.timing](https://developer.mozilla.org/en-US/docs/Web/API/PerformanceTiming)。

虽然不是强制要求，但是特定的时间名称应该作用在大量的span示例上：因为追踪系统可能使用这些事件名称（包括时间戳），对span进行分析汇总。更多信息，可参考[Data Conventions Guidelines 数据约定指南](api/data-conventions.html)。

<span id="tags"></span>

### Tags

每个span可以有多个键值对（key:value）形式的**Tags**，**Tags**是没有时间戳的，支持简单的对span进行注解和补充。

和使用**Logs**的场景一样，对于应用程序特定场景已知的键值对**Tags**，tracer可以对他们特别关注一下。更多信息，可参考[Data Conventions Guidelines 数据约定指南](api/data-conventions.html)。

## SpanContext

每个span必须提供方法访问**SpanContext**。SpanContext代表跨越进程边界，传递到下级span的状态。(例如，包含`<trace_id, span_id, sampled>`元组)，并用于封装**Baggage** (关于Baggage的解释，请参考下文)。SpanContext在跨越进程边界，和在追踪图中创建边界的时候会使用。(ChildOf关系或者其他关系，参考[Span间关系](#Span间关系) )。

### Baggage

**Baggage**是存储在SpanContext中的一个键值对(SpanContext)集合。它会在一条追踪链路上的所有span内_全局_传输，包含这些span对应的SpanContexts。在这种情况下，"Baggage"会随着trace一同传播，他因此得名（Baggage可理解为随着trace运行过程传送的行李）。鉴于全栈OpenTracing集成的需要，Baggage通过透明化的传输任意应用程序的数据，实现强大的功能。例如：可以在最终用户的手机端添加一个Baggage元素，并通过分布式追踪系统传递到存储层，然后再通过反向构建调用栈，定位过程中消耗很大的SQL查询语句。

Baggage拥有强大功能，也会有很大的_消耗_。由于Baggage的全局传输，如果包含的数量量太大，或者元素太多，它将降低系统的吞吐量或增加RPC的延迟。

## Baggage vs. Span Tags

- Baggage在全局范围内，跨进程传输（并且包含实际的应用数据）。Span的tag不会进行传输，因为他们不会被子级的span继承。
- span的tag记录业务系统之外的数据，并存储于追踪系统中。实现OpenTracing时，可以选择用Baggage来存储非业务数据.（TODO：out-of-band and in-band）
- Span Tags are recorded out-of-band from the application data, presumably in the tracing system's storage. Implementations may choose to also record Baggage out-of-band, though that decision is not dictated by the OpenTracing specification.

## Inject and Extract

SpanContexts可以通过**Injected**操作向**Carrier**增加，或者通过**Extracted**从**Carrier**中获取，跨进程通讯数据（例如：HTTP头）。通过这种方式，SpanContexts可以跨越进程边界，并提供足够的信息来建立跨进程的span间关系（因此可以实现跨进程连续追踪）。

# 平台无关的API语义

OpenTracing支持了很多不同的平台，当然，每个平台的API试图保持各平台和语言的习惯和管理，尽量做到入乡随俗。也就是说，每个平台的API，都需要根据上述的核心tracing概念来建模实现。在这一章中，我们试图描述这些概念和语义，尽量减少语言和平台的影响。

## The `Span` Interface

`Span`接口必须实现以下的功能：

- **Get the `Span`'s [`SpanContext`](#SpanContext)**， 通过span获取SpanContext （即使`span`已经结束，或者即将结束）
- **Finish**，完成已经开始的`Span`。处理获取`SpanContext`之外，Finish必须是span实例的最后一个被调用的方法。**(py: `finish`, go: `Finish`)**。 一些的语言实现方法会在`Span`结束之前记录相关信息，因为Finish方法可能不会被调用，因为主线程处理失败或者其他程序错误。在这种情况下，实现应该明确的记录`Span`，保证数据的持久化。
- **Set a key:value tag on the `Span`.**，为`Span`设置tag。tag的key必须是`string`类型，value必须是`string`，`boolean`或数字类型。tag其他类型的value是没有定义的。如果多个value对应同一个key（例如被设置多次），实现方式是没有被定义的。**(py: `set_tag`, go: `SetTag`)**
- **Add a new log event**，为`Span`增加一个log事件。事件名称是`string`类型，参数值可以是任何类型，任何大小。tracer的实现者不一定保存所有的参数值（设置可以所有参数值都不保存）。其中的时间戳参数，可以设置当前时间之前的时间戳。**(py: `log`, go: `Log`)**

## The `SpanContext` Interface

`SpanContext`接口必须实现以下功能。用户可以通过`Span`实例或者`Tracer`的Extract能力获取`SpanContext`接口实例。

- **Set a Baggage item**, 设置一个string:string类型的键值对。注意，新设置的Baggage元素，只保证传递到未来的子级的`Span`。参考下图所示。**(py: `set_baggage_item`, go: `SetBaggageItem`)**
- **Get a Baggage item**， 通过key获取Baggage中的元素。**(py: `get_baggage_item`, go: `BaggageItem`)**

~~~
        [Span A]
            |
     +------+------+
     |             |
 [Span B]      [Span C] ←←← (1) BAGGAGE ITEM "X" IS SET ON SPAN C,BUT AFTER SPAN E ALREADY STARTED.
     |             |            为SPAN C设置BAGGAGE元素，值为X，时间点为SPAN E已经开始运行
 [Span D]      +---+-----+
               |         |
           [Span E]  [Span F] >>> [Span G] >>> [Span H]
                                                 ↑
                                                 ↑
                                                 ↑
             (2) BAGGAGE ITEM "X" IS AVAILABLE FOR RETRIEVAL BY SPAN H (A CHILD OF SPAN C), AS WELL AS SPANS F AND G.
                 SPAN C元素的F\G\H子级span可以读取到BAGGAGE的元素X
~~~

- 虽然以前`SpanContext`是`Tracer`接口的一部分，但是`SpanContext`对于[Inject and Extract](#inject-extract)是必不可少的。

## The `Tracer` Interface

`Tracer`接口必须实现以下功能：

- **Start a new `Span`**. The caller can specify zero or more [`SpanContext` references](#references) (e.g., `FollowsFrom` or `ChildOf`), an explicit start timestamp (other than "now"), and an initial set of `Span` tags. **(py: `start_span`, go: `StartSpan`)**
- <span id="inject-extract"></span>**Inject a `SpanContext`** into a "carrier" object for cross-process propagation. The type of the carrier is either determined through reflection or an explicit [format identifier](/propagation#format-identifiers). See the [end-to-end propagation example](/propagation#propagation-example) to make this more concrete.
- **Extract a `SpanContext`** given a "carrier" object whose contents crossed a process boundary. Extract examines the carrier and tries to reconstruct the previously-Injected `SpanContext` instance. Unless there's an error, Extract returns a `SpanContext` which can be used in the host process like any other, most likely to start a new (child) Span. (Note that some OpenTracing implementations consider the `Span`s on either side of an RPC to have the same identity, and others consider the client to be the parent and the server to be the child) The type of the carrier is either determined through language reflection or an explicit [format](api/cross-process-tracing#format-identifiers). See the [end-to-end propagation example](api/cross-process-tracing#propagation-example) to make this more concrete.

### Global and No-op Tracers

A per-platform OpenTracing API library (e.g., [opentracing-go](https://github.com/opentracing/opentracing-go), [opentracing-java](https://github.com/opentracing/opentracing-java), etc; not OpenTracing `Tracer` *implementations*) must provide a no-op Tracer as part of their interface. The no-op Tracer implementation must not crash and should have no side-effects, including baggage propagation. The Tracer implementation must provide a no-op Span implementation as well; in this way, the instrumentation code that relies on Span instances returned by the Tracer does not need to change to accommodate the possibility of no-op implementations. The no-op Tracer's Inject method should always succeed, and Extract should always behave as if no `SpanContext` could be found in the carrier.

The per-platform OpenTracing API libraries *may* provide support for configuring (Go: `InitGlobalTracer()`, py: `opentracing.tracer = myTracer`) and retrieving (Go: `GlobalTracer()`, py: `opentracing.tracer`) a global/singleton Tracer instance. If a global Tracer is supported, the default must be the no-op Tracer.
