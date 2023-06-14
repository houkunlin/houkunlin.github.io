---
title: Spring Webclient 请求Body限制 256KB 问题
date: 2023-06-14 16:19:16
updated: 2023-06-14 16:19:16
tags:
---

这个问题很久之前遇到过了，这次遇到另外一个有点类似的问题，那就是 SpringCloudGateway 请求限制 256KB 问题，在这里也记录一下 WebClient 请求限制 256KB 问题的处理方式。

在 [How to Resolve Spring Webflux DataBufferLimitException](https://www.baeldung.com/spring-webflux-databufferlimitexception) 这篇文章中有提到解决方案，在这里复制一下以做记录。

在创建 `WebClient` 时设置 `exchangeStrategies` 参数，示例代码如下：

```java
public class Test {
    // 全局
    @Bean
    public WebClient getProgSelfWebClient() {
        // 按照原文档说明，此方式好像还需要增加配置文件问题： spring.codec.max-in-memory-size=500KB
        return WebClient.builder()
          .baseUrl(host)
          .exchangeStrategies(
            ExchangeStrategies.builder()
	          .codecs(codecs -> codecs.defaultCodecs().maxInMemorySize(500 * 1024))
	          .build()
           )
          .build();
    }
    
    // 手动创建，争对单个 Client 接口生效
    @Bean
    public FileHttpClient remoteDemoLoadBalancer(final ReactorLoadBalancerExchangeFilterFunction reactorLoadBalancerExchangeFilterFunction) {
        // 配置默认缓冲区大小，否则默认 256K
        final ExchangeStrategies strategies = ExchangeStrategies.builder()
                .codecs(codecs -> codecs.defaultCodecs().maxInMemorySize(-1))
                .build();

        WebClient client = WebClient.builder()
                .exchangeStrategies(strategies)
                .filter(reactorLoadBalancerExchangeFilterFunction)
                .baseUrl("http://files-server/")
                .build();
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builder(WebClientAdapter.forClient(client)).build();
        return factory.createClient(FileHttpClient.class);
    }
}
```



[1]: https://www.baeldung.com/spring-webflux-databufferlimitexception	"How to Resolve Spring Webflux DataBufferLimitException"



