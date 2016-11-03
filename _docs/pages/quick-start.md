# **10分钟完成分布式追踪**

*_原始发布地址 [OpenTracing blog](https://medium.com/opentracing/distributed-tracing-in-10-minutes-51b378ee40f1#.ypcuah408)_*

随着并发和移除成为现代软件应用的必然特性，分布式追踪系统成为有效监控的一个必须的组成部分。尽管如此，监控并追踪一个系统的调用情况，至今仍是一个耗时而复杂的任务。随着系统的调用分布式程度（超过10个进程）和并发度越来越高，移动端与web端、客户端到服务端的调用关系越来越复杂，追踪调用关系带来的好处是显而易见的。但是选择和部署一个追踪系统的过程十分复杂。[OpenTracing](http://opentracing.io/)标准将改变这一点，OpenTracing尽力让监控一个分布式调用过程简单化。正如我下面视频演示的那样，你能在10分钟内快速配置一个监控系统。

![image alt text](/images/QS_01.gif)

本文描述的示例应用程序使用过程截图

试想一个简单的web网站。当用户访问你的首页时，web服务器发起两个HTTP调用，其中每个调用又访问了数据库。这个过程是否简单直白，发现请求缓慢的原因都不会太复杂。如果你考虑到调用延迟，你可以为每个调用分布式唯一的ID，并通过HTTP头进行传递。如果请求耗时过长，你通过使用唯一ID来grep日志文件，发现问题出在哪里。现在，想想一下，你的web网站变得流行起来，你开始使用分布式架构，你的应用需要跨越多个机器，多个服务来工作。随着机器和服务数量的增长，日志文件能明确解决问题的机会越来越少。确定问题发生的原因将越来越困难。这时，你发现投入调用流程追踪能力是非常有价值的。

正如我提到的，OpenTracing因为[standardizes instrumentation, 监控标准化](https://medium.com/opentracing/towards-turnkey-distributed-tracing-5f4297d1736)，会使得追踪过程变得容易。它意味着，你可以先进行追踪，再决定最终的实现方案。

以[AppDash](https://github.com/sourcegraph/appdash)为例，你可以根据如下的步骤，从编译web项目到查看追踪信息。或者，你可以直接使用Appdash来完成追踪并查看追踪信息。

```
docker run --rm -ti -p 8080:8080 -p 8700:8700 bg451/opentracing-example
```

这将启动一个测试的本地的Appdash实例。[点击](https://github.com/bg451/opentracing-example)查看源码

如果你想看到完成的实例，你可以根据下面的步骤，自己构建webapp，使用OpenTracing设置追踪，绑定到一个追踪系统（如AppDash），并最终查看调用情况。

## **创建一个web工程**

在开始之前，先写几个简单的调用点:

```go
// Acts as our index page
func indexHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte(`<a href="/home"> Click here to start a request </a>`))
}
func homeHandler(w http.ResponseWriter, r *http.Request) {
     w.Write([]byte("Request started"))
    go func() {
        http.Get("http://localhost:8080/async")
    }()
    http.Get("http://localhost:8080/service")
    time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
    w.Write([]byte("Request done!"))
}
// Mocks a service endpoint that makes a DB call
func serviceHandler(w http.ResponseWriter, r *http.Request) {
    // ...
    http.Get("http://localhost:8080/db")
    time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
    // ...
}
// Mocks a DB call
func dbHandler(w http.ResponseWriter, r *http.Request) {
    time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
    // here would be the actual call to a DB.
}
```

将这些调用点组合成一个server

```go
func main() {
    port := 8080
    addr := fmt.Sprintf(":%d", port)
    mux := http.NewServeMux()
    mux.HandleFunc("/", indexHandler)
    mux.HandleFunc("/home", homeHandler)
    mux.HandleFunc("/async", serviceHandler)
    mux.HandleFunc("/service", serviceHandler)
    mux.HandleFunc("/db", dbHandler)
    fmt.Printf("Go to http://localhost:%d/home to start a request!\n", port)
    log.Fatal(http.ListenAndServe(addr, mux))
}
```

将这些放到`main.go`文件中，运行`go run main.go`。

#### **监控应用程序**

现在，你有了一个可以工作的web应用服务器，你可以开始监控它了。你可以开始像下面这样，在入口设置一个span：

```go
func homeHandler(w http.ResponseWriter, r *http.Request) {
  span := opentracing.StartSpan("/home") // Start a span using the global, in this case noop, tracer
  defer span.Finish()
  // ... the rest of the function
}
```

这个span记录**homeHandler**方法完成所需的时间，这只是可以记录的信息的冰山一角。OpenTracing允许你为每一个span设置[tags](spec/#tags)和[logs](spec/#logs)。例如：你可以通过**homeHandler**方法是否正确返回，决定是否记录方法调用的错误信息：

```go
// The ext package provides a set of standardized tags available for use.
import "github.com/opentracing/opentracing-go/ext"

func homeHandler(w http.ResponseWriter, r *http.Request) {
// ...
// We record any errors now.
    _, err := http.Get("http://localhost:8080/service")
    if err != nil {
        ext.Error.Set(span, true) // Tag the span as errored
        span.LogEventWithPayload("GET service error", err) // Log the error
    }
// ...
}
```

你也可以其他其他信息，如：发生的重要事件，用户id，浏览器类型。

然而，这只是其中的一个功能。为了构建真正的端到端追踪，你需要包含调用HTTP请求的客户端的span信息。在我们的示例中，你需要在端到端过程中传递span的上下文信息，使得各端中的span可以合并到一个追踪过程中。这就是API中Inject/Extract的职责。**homeHandler**方法在第一次被调用时，创建一个根span，后续过程如下：

```go
func homeHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Request started"))
    span := opentracing.StartSpan("/home")
    defer span.Finish()

    // Since we have to inject our span into the HTTP headers, we create a request
    asyncReq, _ := http.NewRequest("GET", "http://localhost:8080/async", nil)
    // Inject the span context into the header
    err := span.Tracer().Inject(span.Context(),
        opentracing.TextMap,
        opentracing.HTTPHeaderTextMapCarrier(asyncReq.Header))
    if err != nil {
        log.Fatalf("Could not inject span context into header: %v", err)
    }
    go func() {
        if _, err := http.DefaultClient.Do(asyncReq); err != nil {
            span.SetTag("error", true)
            span.LogEvent(fmt.Sprintf("GET /async error: %v", err))
        }
    }()
    // Repeat for the /service call.
    // ....
}
```

上述代码，在底层实际的执行逻辑是：将关于本地追踪调用的span的元信息，被设置到http的头上，并准备传递出去。下面会展示如何在**serviceHandler**服务中提取这个元数据信息。

```go
func serviceHandler(w http.ResponseWriter, r *http.Request) {
    var sp opentracing.Span
    opName := r.URL.Path
    // Attempt to join a trace by getting trace context from the headers.
    wireContext, err := opentracing.GlobalTracer().Extract(
        opentracing.TextMap,
        opentracing.HTTPHeaderTextMapCarrier(r.Header))
    if err != nil {
        // If for whatever reason we can't join, go ahead an start a new root span.
        sp = opentracing.StartSpan(opName)
    } else {
        sp = opentracing.StartSpan(opName, opentracing.ChildOf(wireContext))
    }
  defer sp.Finish()
  // ... rest of the function
```

如上述程序所示，你可以通过http头获取元数据。你可以重复此步骤，为你需要追踪的调用进行设置，很快，你将可以监控整套系统。如何决定哪些调用需要被追踪呢？你可以考虑你的调用的关键路径。

## **Connect the tracer**

One of the great things about OpenTracing is that once your system is instrumented, adding a tracer is really straightforward! In this example, you can see that I’ve used Appdash, an open source tracing system. There’s small chunk of code needed inside your main function to start the Appdash instance. However, you won’t need to touch any of your instrumentation code at all. In your main function, add:


```goimport (
	"sourcegraph.com/sourcegraph/appdash"
	“sourcegraph.com/sourcegraph/appdash/traceapp”
	appdashot "sourcegraph.com/sourcegraph/appdash/opentracing"
)

func main() {
	// ...
  	store := appdash.NewMemoryStore()

	// Listen on any available TCP port locally.
	l, err := net.ListenTCP("tcp", &net.TCPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 0})
	if err != nil {
		log.Fatal(err)
	}
	collectorPort := l.Addr().(*net.TCPAddr).Port
	collectorAdd := fmt.Sprintf(":%d", collectorPort)

	// Start an Appdash collection server that will listen for spans and
	// annotations and add them to the local collector (stored in-memory).
	cs := appdash.NewServer(l, appdash.NewLocalCollector(store))
	go cs.Start()

	// Print the URL at which the web UI will be running.
	appdashPort := 8700
	appdashURLStr := fmt.Sprintf("http://localhost:%d", appdashPort)
	appdashURL, err := url.Parse(appdashURLStr)
	if err != nil {
		log.Fatalf("Error parsing %s: %s", appdashURLStr, err)
	}
	fmt.Printf("To see your traces, go to %s/traces\n", appdashURL)

	// Start the web UI in a separate goroutine.
	tapp, err := traceapp.New(nil, appdashURL)
	if err != nil {
	 	log.Fatal(err)
	}
	tapp.Store = store
	tapp.Queryer = store
	go func() {
	log.Fatal(http.ListenAndServe(fmt.Sprintf(":%d", appdashPort), tapp))
	}()

	tracer := appdashot.NewTracer(appdash.NewRemoteCollector(collectorPort))
	opentracing.InitGlobalTracer(tracer)
// ...
}
```

This will create an embedded Appdash instance and serve traces locally.

![image alt text](/images/QS_02.png)

Should you want to change your tracer implementation, it is a O(1) change because of OpenTracing. All you need to do is update your main function; the rest of your instrumentation stays the same. For example, if you decide to use Zipkin later on, this is all you would need to do in your main function:

```go
import zipkin "github.com/openzipkin/zipkin-go-opentracing"

func main() {
  // ...
  // Replace Appdash tracer code with this
  collector, err := zipkin.NewKafkaCollector("ZIPKIN_ADDR")
  if err != nil {
    log.Fatal(err)
    return
  }

  tracer, err = zipkin.NewTracer(
    zipkin.NewRecorder(collector, false, "localhost:8000", "example"),
  )
  if err != nil {
    log.Fatal(err)
  }
  opentracing.InitGlobalTracer(tracer)
  // ...
}
```

Having made it thus far, you can see that instrumenting your code for tracing is much easier with OpenTracing. I recommend this as a best practice whenever starting out on an app. That’s because by setting up tracing even when your application is small, trace data can guide your development strategy as you grow. Having visibility into your processes as they start to mature and increase in complexity will help you build a sustainable product.

---
origin/原文

# **Distributed Tracing in 10 Minutes**

*_originally posted on [OpenTracing blog](https://medium.com/opentracing/distributed-tracing-in-10-minutes-51b378ee40f1#.ypcuah408)_*

With the intrinsic concurrency and asynchrony of modern software applications, distributed tracing has become part of the table stakes for effective monitoring. That said, instrumenting a system for tracing has, at least historically, been a labor-intensive, complicated task. Tracing brings the benefits of visibility into an application as it grows to 10+ processes, starts seeing increased concurrency, or non-trivial interactions between mobile/web clients and servers. But setting up instrumentation and deciding which tracer to use can add up to a large project. The [OpenTracing](http://opentracing.io/) standard changes that, making it possible to instrument applications for distributed tracing with minimal effort. As I demonstrate below, you can now easily set up tracing in less than 10 minutes with OpenTracing.

![image alt text](/images/QS_01.gif)

A screencast of the example app built in this post

Imagine a simple website. Whenever a user goes to your home page, the web server makes two HTTP calls, and each of those calls branches out and makes a call to the database. This is fairly straightforward and debugging any slow requests wouldn’t be too difficult. If you are serious about latency, you might assign each request a unique ID and propagate it downstream through HTTP headers. If a request took a long time, you can then grep over the log files for that request ID to figure out what was going on. Now imagine your website starts becoming popular and your application is spread across multiple machines and services. As the number of machines and services grow, logs provide less and less visibility. Determining causality gets tricky pretty quickly. This is when you realize workflow tracing would have more than worth the investment.

As I mentioned, OpenTracing steps in to make it very easy for you to trace because it [standardizes instrumentation](https://medium.com/opentracing/towards-turnkey-distributed-tracing-5f4297d1736). What that means is that you can instrument first and defer most implementation decisions to later.

You can follow my entire process below — from building the web app to seeing the traces in [AppDash](https://github.com/sourcegraph/appdash), the open source distributed tracing system I chose. Alternatively, you can skip ahead and see the finished result with Appdash. To do that, run

```
docker run --rm -ti -p 8080:8080 -p 8700:8700 bg451/opentracing-example
```

This will spin up the test server and a local Appdash instance. The source code can be found at [here](https://github.com/bg451/opentracing-example).

For those who want to see the full story, you can go through the full exercise of building the web app, instrumenting it with OpenTracing, binding to a tracer, AppDash, and finally seeing the traces, in this blog post.

## **Building the web app**

To start off, write a few simple endpoints:

```go
// Acts as our index page
func indexHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte(`<a href="/home"> Click here to start a request </a>`))
}
func homeHandler(w http.ResponseWriter, r *http.Request) {
     w.Write([]byte("Request started"))
    go func() {
        http.Get("http://localhost:8080/async")
    }()
    http.Get("http://localhost:8080/service")
    time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
    w.Write([]byte("Request done!"))
}
// Mocks a service endpoint that makes a DB call
func serviceHandler(w http.ResponseWriter, r *http.Request) {
    // ...
    http.Get("http://localhost:8080/db")
    time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
    // ...
}
// Mocks a DB call
func dbHandler(w http.ResponseWriter, r *http.Request) {
    time.Sleep(time.Duration(rand.Intn(200)) * time.Millisecond)
    // here would be the actual call to a DB.
}
```

Put this all together into a working server.

```go
func main() {
    port := 8080
    addr := fmt.Sprintf(":%d", port)
    mux := http.NewServeMux()
    mux.HandleFunc("/", indexHandler)
    mux.HandleFunc("/home", homeHandler)
    mux.HandleFunc("/async", serviceHandler)
    mux.HandleFunc("/service", serviceHandler)
    mux.HandleFunc("/db", dbHandler)
    fmt.Printf("Go to http://localhost:%d/home to start a request!\n", port)
    log.Fatal(http.ListenAndServe(addr, mux))
}
```

Throw everything into `main.go` file and run: `go run main.go`.

#### **Instrument the app**

Now that you have a working web server, you can start instrumenting it. Start at the top level and work your way down. You can start a span and finish it like so:

```go
func homeHandler(w http.ResponseWriter, r *http.Request) {
  span := opentracing.StartSpan("/home") // Start a span using the global, in this case noop, tracer
  defer span.Finish()
  // ... the rest of the function
}
```

This span records how long it takes **homeHandler** to complete, but that’s just the tip of the iceberg in terms of information that you can record. OpenTracing enables you to attach [tags](spec/#tags) and [logs](spec/#logs) to an individual span. For instance, you can specify whether or not a span contains an error inside **homeHandler**:

```go
// The ext package provides a set of standardized tags available for use.
import "github.com/opentracing/opentracing-go/ext"

func homeHandler(w http.ResponseWriter, r *http.Request) {
// ...
// We record any errors now.
    _, err := http.Get("http://localhost:8080/service")
    if err != nil {
        ext.Error.Set(span, true) // Tag the span as errored
        span.LogEventWithPayload("GET service error", err) // Log the error
    }
// ...
}
```

You can record other things as well, including important events, the user ID, and the browser type.

However, that’s only for one function. To build true end-to-end traces, you’ll want to include spans for the client side of the HTTP calls. In our example, you need to start propagating span contexts downstream to the other endpoints now, and those endpoints need to be able to join traces. This is where the Inject/Extract part of the API comes into play. **homeHandler** creates a "root" span since it’s the first thing to get called. You will start there and work your way down.

```go
func homeHandler(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("Request started"))
    span := opentracing.StartSpan("/home")
    defer span.Finish()

    // Since we have to inject our span into the HTTP headers, we create a request
    asyncReq, _ := http.NewRequest("GET", "http://localhost:8080/async", nil)
    // Inject the span context into the header
    err := span.Tracer().Inject(span.Context(),
        opentracing.TextMap,
        opentracing.HTTPHeaderTextMapCarrier(asyncReq.Header))
    if err != nil {
        log.Fatalf("Could not inject span context into header: %v", err)
    }
    go func() {
        if _, err := http.DefaultClient.Do(asyncReq); err != nil {
            span.SetTag("error", true)
            span.LogEvent(fmt.Sprintf("GET /async error: %v", err))
        }
    }()
    // Repeat for the /service call.
    // ....
}
```

What happens underneath is that the underlying implementation injects a span’s metadata about the current trace into the request’s headers to be read by anyone downstream. Go ahead and extract that data in **serviceHandler**.

```go
func serviceHandler(w http.ResponseWriter, r *http.Request) {
    var sp opentracing.Span
    opName := r.URL.Path
    // Attempt to join a trace by getting trace context from the headers.
    wireContext, err := opentracing.GlobalTracer().Extract(
        opentracing.TextMap,
        opentracing.HTTPHeaderTextMapCarrier(r.Header))
    if err != nil {
        // If for whatever reason we can't join, go ahead an start a new root span.
        sp = opentracing.StartSpan(opName)
    } else {
        sp = opentracing.StartSpan(opName, opentracing.ChildOf(wireContext))
    }
  defer sp.Finish()
  // ... rest of the function
```

And that’s it! If you repeat the steps above for things you want to trace, you should have a fully instrumented system fairly quickly. To decide what needs be traced, you should look at your requests’ critical paths.

## **Connect the tracer**

One of the great things about OpenTracing is that once your system is instrumented, adding a tracer is really straightforward! In this example, you can see that I’ve used Appdash, an open source tracing system. There’s small chunk of code needed inside your main function to start the Appdash instance. However, you won’t need to touch any of your instrumentation code at all. In your main function, add:

```go
import (
	"sourcegraph.com/sourcegraph/appdash"
	“sourcegraph.com/sourcegraph/appdash/traceapp”
	appdashot "sourcegraph.com/sourcegraph/appdash/opentracing"
)

func main() {
	// ...
  	store := appdash.NewMemoryStore()

	// Listen on any available TCP port locally.
	l, err := net.ListenTCP("tcp", &net.TCPAddr{IP: net.IPv4(127, 0, 0, 1), Port: 0})
	if err != nil {
		log.Fatal(err)
	}
	collectorPort := l.Addr().(*net.TCPAddr).Port
	collectorAdd := fmt.Sprintf(":%d", collectorPort)

	// Start an Appdash collection server that will listen for spans and
	// annotations and add them to the local collector (stored in-memory).
	cs := appdash.NewServer(l, appdash.NewLocalCollector(store))
	go cs.Start()

	// Print the URL at which the web UI will be running.
	appdashPort := 8700
	appdashURLStr := fmt.Sprintf("http://localhost:%d", appdashPort)
	appdashURL, err := url.Parse(appdashURLStr)
	if err != nil {
		log.Fatalf("Error parsing %s: %s", appdashURLStr, err)
	}
	fmt.Printf("To see your traces, go to %s/traces\n", appdashURL)

	// Start the web UI in a separate goroutine.
	tapp, err := traceapp.New(nil, appdashURL)
	if err != nil {
	 	log.Fatal(err)
	}
	tapp.Store = store
	tapp.Queryer = store
	go func() {
	log.Fatal(http.ListenAndServe(fmt.Sprintf(":%d", appdashPort), tapp))
	}()

	tracer := appdashot.NewTracer(appdash.NewRemoteCollector(collectorPort))
	opentracing.InitGlobalTracer(tracer)
// ...
}
```

This will create an embedded Appdash instance and serve traces locally.

![image alt text](/images/QS_02.png)

Should you want to change your tracer implementation, it is a O(1) change because of OpenTracing. All you need to do is update your main function; the rest of your instrumentation stays the same. For example, if you decide to use Zipkin later on, this is all you would need to do in your main function:

```go
import zipkin "github.com/openzipkin/zipkin-go-opentracing"

func main() {
  // ...
  // Replace Appdash tracer code with this
  collector, err := zipkin.NewKafkaCollector("ZIPKIN_ADDR")
  if err != nil {
    log.Fatal(err)
    return
  }

  tracer, err = zipkin.NewTracer(
    zipkin.NewRecorder(collector, false, "localhost:8000", "example"),
  )
  if err != nil {
    log.Fatal(err)
  }
  opentracing.InitGlobalTracer(tracer)
  // ...
}
```

Having made it thus far, you can see that instrumenting your code for tracing is much easier with OpenTracing. I recommend this as a best practice whenever starting out on an app. That’s because by setting up tracing even when your application is small, trace data can guide your development strategy as you grow. Having visibility into your processes as they start to mature and increase in complexity will help you build a sustainable product.
