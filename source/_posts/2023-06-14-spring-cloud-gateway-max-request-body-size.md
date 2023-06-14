---
title: Spring Cloud Gateway 上传文件大小限制 256KB 问题、请求Body限制 256KB 问题
date: 2023-06-14 15:43:38
updated: 2023-06-14 15:43:38
tags:
---



## 解决常规请求大小限制问题

近日，在把文件上传接口放在 SpringCloudGateway 后发现偶尔上传文件失败问题，经过初步排查前端上传文件超过256KB就会出错，经过简单日志排查，发现是我们的网关问题，错误日志大致如下：

```text
org.springframework.core.io.buffer.DataBufferLimitException: Exceeded limit on max bytes to buffer : 262144
	at org.springframework.core.io.buffer.LimitedDataBufferList.raiseLimitException(LimitedDataBufferList.java:99)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException:
Error has been observed at the following site(s):
	*__checkpoint ⇢ org.springframework.cloud.gateway.filter.WeightCalculatorWebFilter [DefaultWebFilterChain]
	*__checkpoint ⇢ org.springframework.web.filter.reactive.ServerHttpObservationFilter [DefaultWebFilterChain]
	*__checkpoint ⇢ HTTP POST "/api/files/upload" [ExceptionHandlingWebHandler]
Original Stack Trace:
		at org.springframework.core.io.buffer.LimitedDataBufferList.raiseLimitException(LimitedDataBufferList.java:99)
		at org.springframework.core.io.buffer.LimitedDataBufferList.updateCount(LimitedDataBufferList.java:92)
		at org.springframework.core.io.buffer.LimitedDataBufferList.add(LimitedDataBufferList.java:58)
		at reactor.core.publisher.MonoCollect$CollectSubscriber.onNext(MonoCollect.java:103)
		at reactor.core.publisher.FluxMap$MapSubscriber.onNext(FluxMap.java:122)
// .......
2023-06-14T10:54:34.710+08:00 ERROR 54904 --- [ctor-http-nio-6] r.n.channel.ChannelOperationsHandler     : [5f5eda41-1, L:/127.0.0.1:8081 ! R:kubernetes.docker.internal/127.0.0.1:55331] Error was received while reading the incoming data. The connection will be closed.

io.netty.util.IllegalReferenceCountException: refCnt: 0, decrement: 1
	at io.netty.util.internal.ReferenceCountUpdater.toLiveRealRefCnt(ReferenceCountUpdater.java:83)
	at io.netty.util.internal.ReferenceCountUpdater.release(ReferenceCountUpdater.java:148)
	at io.netty.buffer.AbstractReferenceCountedByteBuf.release(AbstractReferenceCountedByteBuf.java:101)
	at io.netty.handler.codec.http.DefaultHttpContent.release(DefaultHttpContent.java:92)
	at io.netty.util.ReferenceCountUtil.release(ReferenceCountUtil.java:90)
	at reactor.netty.channel.FluxReceive.onInboundNext(FluxReceive.java:380)
```

发现是网关默认限制了请求Body的大小为 256KB，而我们则需要修改这个限制，经过网上查找资料，找到 [How to Resolve Spring Webflux DataBufferLimitException](https://www.baeldung.com/spring-webflux-databufferlimitexception) 这篇文章，文章中的解决方式如下：

实现 `WebFluxConfigurer` 接口并重写 `void configureHttpMessageCodecs(ServerCodecConfigurer)` 方法，

```java
@Configuration
public class WebFluxConfiguration implements WebFluxConfigurer {
    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        // 可以把下面的 500 * 1024 改为 -1 表示不限制请求大小
        configurer.defaultCodecs().maxInMemorySize(500 * 1024);
    }
}
```

以及在配置文件（`application.yml`）中增加配置内容：

```yaml
spring:
  codec:
    # 可以把下面的 500KB 改为 -1 表示不限制请求大小
    max-in-memory-size: 500KB
```

经过以上两个设置，就可以成功修改请求大小限制。



## 解决常规解决方案不生效问题

一般情况下，经过上面的处理，都可以成功的修改Gateway的请求限制，但是有时候他莫名其妙的失效了，在本次的错误处理中，我就遇到了这个问题。

最终经过排查，是我向Gateway增加了一个 `RequestLogFilter` 请求日志打印的全局过滤器导致的，在这个过滤器中，我手动创建了一个 `XxxGatewayFilterFactory` 对象，大致代码如下：

```java
@Component
@RequiredArgsConstructor
public class RequestLogFilter implements GlobalFilter {
    private GatewayFilter delegate = null;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        if (delegate == null) {
            return chain.filter(exchange);
        }
        // .......
        return delegate.filter(exchange, chain);
    }

    private ModifyRequestBodyGatewayFilterFactory.Config getConfig() {
        ModifyRequestBodyGatewayFilterFactory.Config cf = new ModifyRequestBodyGatewayFilterFactory.Config();
        // .......
        return cf;
    }

    @PostConstruct
    public void init() {
        this.delegate = new ModifyRequestBodyGatewayFilterFactory().apply(this.getConfig());
    }
}
```

在上面的 `RequestLogFilter` 对象中，在初始化Bean阶段创建了 `ModifyRequestBodyGatewayFilterFactory` 对象，而在其构造方法中使用 `HandlerStrategies.withDefaults().messageReaders()` 创建了一个默认的 `List<HttpMessageReader<?>>` ，问题就是出在这个地方。

`HandlerStrategies.withDefaults().messageReaders()` 会创建一个默认的 `ServerCodecConfigurer` 对象，此对象负责初始化相关的 `HttpMessageReader` 对象，而这个环节中，`HandlerStrategies` 和 `ServerCodecConfigurer` 都无法读取我们前面配置的 `maxInMemorySize ` 参数，此时 `ServerCodecConfigurer#maxInMemorySize == null`  ，所以在初始化 `HttpMessageReader` 时无法设置正确的请求大小限制配置，并且最终系统启动完毕后，会直接使用此次创建的 `List<HttpMessageReader<?>>` ，直接导致后续所有的请求都无法突破 256KB 的限制。

实际最终的问题是，代码运行的过程中，手动创建 `ServerCodecConfigurer` 比 SpringBoot 初始化的 `ServerCodecConfigurer` 的时间早 ，导致了Gateway最终 `codecConfigurer.getReaders()` 得到的 `List<HttpMessageReader<?>>` 以第一个为准。

因此解决的方法也很简单，那就是使用 SpringBoot 初始化的 `ServerCodecConfigurer` 对象，或者我们创建 `ServerCodecConfigurer` 对象时手动设置其 `maxInMemorySize` 参数。

例如可参考如下解决方案：

```java
@Component
@RequiredArgsConstructor
public class RequestLogFilter implements GlobalFilter {
    private final ServerCodecConfigurer codecConfigurer;
    private GatewayFilter delegate = null;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        if (delegate == null) {
            return chain.filter(exchange);
        }
        // .......
        return delegate.filter(exchange, chain);
    }

    private ModifyRequestBodyGatewayFilterFactory.Config getConfig() {
        ModifyRequestBodyGatewayFilterFactory.Config cf = new ModifyRequestBodyGatewayFilterFactory.Config();
        // .......
        return cf;
    }

    @PostConstruct
    public void init() {
        this.delegate = new ModifyRequestBodyGatewayFilterFactory(codecConfigurer.getReaders()).apply(this.getConfig());
    }
}
```





[1]: https://www.baeldung.com/spring-webflux-databufferlimitexception	"How to Resolve Spring Webflux DataBufferLimitException"

