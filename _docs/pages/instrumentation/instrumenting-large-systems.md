# 如何追踪大规模分布式系统

_在阅读如何使用OpenTracing标准，监控大规模分布式系统之前，确保你已经阅读过[Specification overview](/pages/spec)章节_

## Spans 和它们之间的关系

实现OpenTracing完成分布式追踪的两个基本概念就是_Spans_和_Relationships_(span间关系)：

* **_[Span](/pages/spec#spans)_** 是系统中的一个逻辑工作单元，包含这个工作单元启动时间和执行时间。在一条追踪链路中，各个span与系统中的不同组件有关，并体现这些组件的执行路径。

  ![image of spans in a system](/images/OTHT_0.png)

* **_[Relationships](/pages/spec/#causal-span-references)_** 是span间的连接关系。一个span可以和0-n个组件存在因果关系。这种关系是的各个span被串接起来，并用来帮助定位追踪链路的关键路径。

  ![image of relationships in a system](/images/OTHT_1.png)

你所期待的结束状态，是获取你所有组件的span，以及它们之间的关系。当开始建立你的分布式追踪系统时，最好的方法是从服务框架（如：RPC层）或者其他和复杂执行路径有关的组件开始。

你可以从使用支持OpenTracing标准的服务框架开始 (如：[gRPC](https://github.com/grpc/grpc-go)）。但是，如果你不支持OpenTracing的框架，你可以阅读[IPC/RPC Framework Guide](/pages/instrumentation/instrumenting-frameworks)章节。

## 专注高价值区域

如上面提到的，从RPC层和你的web框架开始构建追踪，是一个好方法。这两部分将包含事务路径中的大部分内容。

下一步，你应该着手在没有被服务框架覆盖的事务路径上。为足够多的组件增加监控，为高价值的事务创建一条关键链路的追踪轨迹。

你监控的首要目标，是基于关键路径上的span，寻找最耗时的操作，为可量化的优化操作提供最重要的数据支持。例如，对于只占用事务时间1%的操作（一个大粒度的span）增加更细粒度的监控，对于你理解端到端的延迟（性能问题）不会有太大意义。

## 先走再跑，逐步提高

如果你正在构建你的跨应用追踪系统实现，使用这套系统建立高价值的关键事务与平衡关键事务和代码覆盖率的概念。最大的价值，在于为关键事务生成端到端的追踪。可视化展现你的追踪结果是非常重要的。它可能帮助你定于那块区域（代码块/系统模块）需要更细粒度的追踪。

一旦你有了端到端的监控，你很容易评估在哪些区域增加投入，进行更细粒度的追踪，并能确定事情的优先级。如果你开始深入处理监控问题，可以考虑哪些部分能够复用。通过这些复用建立一套可以在多个服务间服用的监控类库。

这种方法可以提供广泛的覆盖（如：RPC，web框架等），也能为关键业务的事务增加高价值的埋点。即使有些埋点（生成span）的代码是一次性工作，也能通过这种模式发现未来工作的优先级，优化工作效率。

## 示例实例

下面的例子让上述的概念更具体一些：

在这个例子中，我们想追踪一个，由手机端发起，调用了多个服务的调用链。

1. 首先，我们必须说明这个事务的大体情况。在我们的例子中，事务如下所示：

  ![image showing a system transaction](/images/OTHT_2.png)

  **_一个客户通过手机客户端向web发起了一个HTTP请求，产生一个复杂的调用流程：mobile client (HTTP) → web tier (RPC) → auth service (RPC) → billing service (RPC) → resource request (API) → response to web tier (API) → response to client (HTTP)_**

2. 现在，我们对事务的大概情况了解，我们去监控一些通用的协议和框架。最好的选择是从RPC服务框架开始，这将是收集web请求背后发生的调用情况的最好方式。（或者说，任何在分布式过程中发生的问题，都会在直接体现在RPC服务中）

3. The next component that makes sense to instrument is the web framework. By adding the web framework we are able to then have an end-to-end trace. It may be rough, but at least the full workflow can be captured by our tracing system.

  ![image of a high-level trace](/images/OTHT_3.png)

4. At this point we want to take a look at the trace and evaluate where our efforts will provide the greatest value. In our example we can see the area that has the greatest delay for this request is the time it takes for resources to be allocated. This is likely a good place to start adding more granular visibility into the allocation process and instrument the components involved. Once we instrument the resource API we see that resource request can be broken down into:

  ![image of a mid-level trace showing a serialized process](/images/OTHT_4.png)
  **_resource request (API) → container startup (API) → storage allocation (API) → startup scripts (API) → resource ready response (API)_**

5. Once we have instrumented the resource allocation process components, we can see that the bulk of the time is during resource provisioning. The next step would be to go a bit deeper and look to see if there were optimizations that could be done in this process. Maybe we could provision resources in parallel rather than in serial.

  ![image of a mid-level trace showing a parallelized process](/images/OTHT_5.png)

6. Now that there is visibility and an established baseline for an end-to-end workflow, we can articulate a realistic external SLO for that request. In addition, articulating SLO’s for internal services can also become part of the conversation about uptime and error budgets.

7. The next iteration would be to go back to the top level of the trace and look for other large spans that appear to lack visibility and apply more granular instrumentation. If instead the visibility is sufficient, the next step would be to move on to another transaction.

8. Rinse and repeat.
