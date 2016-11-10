# Supported Tracer Implementations, 符合标准的项目

## Zipkin and Jaeger

Zipkin 和 Jaeger 在多种语言环境中支持OpenTracing. 另外，还有一个实验性的项目[bridge from Brave (Zipkin Java) instrumentation to OpenTracing](https://github.com/openzipkin/brave-opentracing)。 其他相关链接[zipkin-go-opentracing](https://github.com/openzipkin/zipkin-go-opentracing), [jaeger-client-java](https://github.com/uber/jaeger-client-java), [jaeger-client-go](https://github.com/uber/jaeger-client-go), [jaeger-client-python](https://github.com/uber/jaeger-client-python), and [jaeger-client-node](https://github.com/uber/jaeger-client-node)。


## Appdash

Appdash ([background reading](https://sourcegraph.com/blog/announcing-appdash-an-open-source-perf-tracing/)) 是一个来源自[sourcegraph](https://sourcegraph.com/)的轻量级，基于Golang的分布式追踪系统。有一个兼容OpenTracing的追踪系统实现，使用Appdash作为后端，通过绑定Appdash到OpenTracing监控上，应用系统就可以轻松实现监控：

```go
import (
    "github.com/sourcegraph/appdash"
    appdashtracer "github.com/sourcegraph/appdash/opentracing"
)

func main() {
    // Initialization with a local collector:
    collector := appdash.NewLocalCollector(myAppdashStore)
    chunkedCollector := appdash.NewChunkedCollector(collector)
    tracer := appdashtracer.NewTracer(chunkedCollector)

    // Initialization with a remote collector:
    collector := appdash.NewRemoteCollector("localhost:8700")
    tracer := appdashtracer.NewTracer(collector)
}
```

查看更多信息，可以查阅 [the godocs](https://godoc.org/github.com/sourcegraph/appdash/opentracing).


## LightStep

[LightStep](http://lightstep.com/) 用来在生产环境运行，使用本地化，符合OpenTracing标准的tracer运行一个私有的测试版本。兼容OpenTracing的[LightStep Tracers](https://github.com/lightstep)所支持的语言包括Go, Python, Javasrcipt, Objective-C, Java, PHP, Ruby, and C++.
