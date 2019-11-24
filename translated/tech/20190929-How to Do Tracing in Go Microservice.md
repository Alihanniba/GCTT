在微服务中，调用链是很长且复杂的，为了便于了解一些模块的性能，你需要分布式追踪。这个想法很简单，具体就是，在请求的开始，构造一个唯一的 ID，然后在整个调用链中都携带它。这个 ID 叫做 [Correlation ID](https://hilton.org.uk/blog/microservices-correlation-id)¹，你可以用它来追踪整个请求的性能。这里你需要解决两个问题。第一个，怎么使 ID 在追踪应用内的你需要的每一个方法里都是可用的；第二个，当你需要调用其他微服务时，怎么通过 ID 来传递。

# OpenTracing 是什么？

有很多追踪库，其中最受欢迎的几个大概是 “[Zipkin](https://zipkin.io/)”²  和 “[Jaeger](https://www.jaegertracing.io/)”³。你会选择哪一个呢？这是一个经常会头疼的问题，因为你会选择当下最流行的那个，但是不久又会有更流行的冒出来。 [OpenTracing](https://opentracing.io/docs/getting-started/)⁴ 尝试通过为追踪库创建一个公有的 interface 来解决这一问题，所以以后你可以选择不同的执行库而不用修改你的代码。

# 怎么追踪服务器端点？

在我们的程序里，我们使用 “Zipkin” 作为追踪库，“OpenTracing” 作为公共的追踪 API。在追踪系统里通常有4个组件，我用 Zipkin 来举个例子：

- Recorder：记录追踪信息
- Reporter (or collecting agent)：从 recorder 收集数据并发送数据给 UI app
- Tracer：产出跟踪数据
- UI：在图形化 UI 界面上展现追踪数据

![zipkin](https://miro.medium.com/max/657/1*gJAIxNxkGYvfdcsPgmRQXQ.png)

这就是关于 Zipkin 的图形化流程，我是从 “[Zipkin Architecture](https://zipkin.io/pages/architecture.html)”⁵ 拿到的。

这里有两种不同类型的追踪模式，一种是进程内部追踪，另一种是跨进程追踪。我们首先来看看跨进程追踪。

**The client side code:**

我们用一个简单的 gRPC 程序来举个例子，它有客户端和服务端的代码。我们将追踪一个从客户端到服务端并返回的请求。下面的代码在客户端创建一个新的 tracer，首先它创建了 “HTTP Collector” (the agent)，这个 Collector 收集追踪数据并发送给 “Zipkin” UI。“endpointUrl” 就是 “Zipkin” UI 的 URL。然后，它创建了一个 recorder 来记录这个端点上的信息，“hostUrl” 是 gRPC (client) 回调的 URL。接着，它在我们刚刚创建的 recorder 上新建了一个 tracer。最后，它被设成了 “OpenTracing” 的 “GlobalTracer”，它将在整个应用程序里被访问。

```go
const (
	endpoint_url = "http://localhost:9411/api/v1/spans"
	host_url = "localhost:5051"
	service_name_cache_client = "cache service client"
	service_name_call_get = "callGet"
)

func newTracer () (opentracing.Tracer, zipkintracer.Collector, error) {
	collector, err := openzipkin.NewHTTPCollector(endpoint_url)
	if err != nil {
		return nil, nil, err
	}
	recorder :=openzipkin.NewRecorder(collector, true, host_url, service_name_cache_client)
	tracer, err := openzipkin.NewTracer(
		recorder,
		openzipkin.ClientServerSameSpan(true))

	if err != nil {
		return nil,nil,err
	}
	opentracing.SetGlobalTracer(tracer)

	return tracer,collector, nil
}
```

下面是 gRPC 的客户端代码。它首先调用了上面提到的创建 tracer 的 “newTrace()” 方法，然后，它创建了一个 gRPC 连接，包含了这个新创建的 tracer。下一步，它用刚刚创建的连接创建了一个缓存服务客户端。最后，它通过 gRPC 客户端对这个缓存服务发起了 “Get” 调用。

```go
key:="123"
	tracer, collector, err :=newTracer()
	if err != nil {
		panic(err)
	}
	defer collector.Close()
	connection, err := grpc.Dial(host_url,
		grpc.WithInsecure(), grpc.WithUnaryInterceptor(otgrpc.OpenTracingClientInterceptor(tracer, otgrpc.LogPayloads())),
		)
	if err != nil {
		panic(err)
	}
	defer connection.Close()
	client := pb.NewCacheServiceClient(connection)
	value, err := callGet(key, client)
```

**Trace and Span:**

在 OpenTracing 中，一个重要的概念叫做 “trace”，它表示一个请求的调用链从开始到结束的标识是 “traceID”。在 trace 之下，有很多 spans，在调用链中有很多阶段，它们用 “spanId” 来标识。每个 span 都有一个父亲，在一个 trace 上全部的spans 构成了一个有指向的 acyclic graph(DAG)。下面是反应 spans 的关系图。我是从 “[The OpenTracing Semantic Specification](https://opentracing.io/specification/)”⁶ 看到的。

![span](https://miro.medium.com/max/651/1*bPVV6JYt8Y3-OViaWRQZHg.png)

