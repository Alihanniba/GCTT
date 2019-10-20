在微服务中，调用链是很长且复杂的，为了便于了解一些模块的性能，你需要分布式追踪。这个想法很简单，具体就是，在请求的开始，构造一个唯一的 ID，然后在整个调用链中都携带它。这个 ID 叫做 [Correlation ID](https://hilton.org.uk/blog/microservices-correlation-id)¹，你可以用它来追踪整个请求的性能。这里你需要解决两个问题。第一个，怎么使 ID 在追踪应用内的你需要的每一个方法里都是可用的；第二个，当你需要调用其他微服务时，怎么通过 ID 来传递。

# OpenTracing 是什么？

有很多追踪库，其中最受欢迎的几个大概是 “[Zipkin](https://zipkin.io/)”²  和 “[Jaeger](https://www.jaegertracing.io/)”³。你会选择哪一个呢？这是一个经常会头疼的问题，因为你会选择当下最流行的那个，但是不久又会有更流行的冒出来。 [OpenTracing](https://opentracing.io/docs/getting-started/)⁴ 尝试通过为追踪库创建一个公有的 interface 来解决这一问题，所以以后你可以选择不同的执行库而不用修改你的代码。