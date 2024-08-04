---
title: Spring Boot Caffeine CacheManager 启用缓存统计功能
date: 2024-08-04 23:22:31
updated: 2024-08-04 23:22:31
tags:
---
昨天我在开发后台管理系统的时候，想开发一个缓存清理功能，2个接口，一个接口获取缓存信息，一个接口清空缓存，用的也是 Caffeine 做缓存。

一开始写的时候没想到要加入监控数据，但是在 debug 的时候发现有个 `com.github.benmanes.caffeine.cache.Cache.stats()` 方法可以获取缓存监控信息，然后就高高兴兴的加入进去了，接口功能都写好后，发现我得到的监控统计数据都是0，然后就 debug 看看为什么为0。

![img.png](img.png)

## 1. 查找问题

刚开始走了弯路，调试到一个`SSSMW` 类似这样的类，然后发现调用了 `com.github.benmanes.caffeine.cache.BoundedLocalCache#statsCounter()` 这样一个方法，而方法的内容是

```java com.github.benmanes.caffeine.cache.BoundedLocalCache#statsCounter() 源码
@Override
public StatsCounter statsCounter() {
  return StatsCounter.disabledStatsCounter();
}
```

这看起来就不对了，这个方法直接写死了是 disabled 的，然后就去网上搜，搜到了这么一篇文章 [《Caffeine CacheManager 如何查看命中率、监控》](https://amos.wang/2020/12/03/notes/frame/sping/cache/CaffeineStat/) ，这篇文章是 2020 年的，距离现在已经快 4 年了，不知道是否正确，文章里面提到了需要重写 `org.springframework.cache.caffeine.CaffeineCacheManager` 对象。

![img_1.png](img_1.png)

我觉得重写的代码侵入太严重了，怕后期难维护，觉得可能还有其他的方法可以实现，然后就尝试去看 SpringBoot 源码，然后就找到了 `CaffeineCacheManager` 的配置方法。

```java CaffeineCacheConfiguration 源码
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Caffeine.class, CaffeineCacheManager.class })
@ConditionalOnMissingBean(CacheManager.class)
@Conditional({ CacheCondition.class })
class CaffeineCacheConfiguration {
	@Bean
	CaffeineCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers customizers,
			ObjectProvider<Caffeine<Object, Object>> caffeine, ObjectProvider<CaffeineSpec> caffeineSpec,
			ObjectProvider<CacheLoader<Object, Object>> cacheLoader) {
		CaffeineCacheManager cacheManager = createCacheManager(cacheProperties, caffeine, caffeineSpec, cacheLoader);
		List<String> cacheNames = cacheProperties.getCacheNames();
		if (!CollectionUtils.isEmpty(cacheNames)) {
			cacheManager.setCacheNames(cacheNames);
		}
		return customizers.customize(cacheManager);
	}
	private CaffeineCacheManager createCacheManager(CacheProperties cacheProperties,
			ObjectProvider<Caffeine<Object, Object>> caffeine, ObjectProvider<CaffeineSpec> caffeineSpec,
			ObjectProvider<CacheLoader<Object, Object>> cacheLoader) {
		CaffeineCacheManager cacheManager = new CaffeineCacheManager();
		setCacheBuilder(cacheProperties, caffeineSpec.getIfAvailable(), caffeine.getIfAvailable(), cacheManager);
		cacheLoader.ifAvailable(cacheManager::setCacheLoader);
		return cacheManager;
	}
	private void setCacheBuilder(CacheProperties cacheProperties, CaffeineSpec caffeineSpec,
			Caffeine<Object, Object> caffeine, CaffeineCacheManager cacheManager) {
		String specification = cacheProperties.getCaffeine().getSpec();
		if (StringUtils.hasText(specification)) {
			cacheManager.setCacheSpecification(specification);
		}
		else if (caffeineSpec != null) {
			cacheManager.setCaffeineSpec(caffeineSpec);
		}
		else if (caffeine != null) {
			cacheManager.setCaffeine(caffeine);
		}
	}
}
```

## 2. 定位问题

刚开始看到了这么一行代码 `return customizers.customize(cacheManager);` 在返回的时候用 `CacheManagerCustomizers` 去处理了 `CaffeineCacheManager` 对象，而它的实现如下：

```java CacheManagerCustomizers 源码
public class CacheManagerCustomizers {
	private final List<CacheManagerCustomizer<?>> customizers;
	public CacheManagerCustomizers(List<? extends CacheManagerCustomizer<?>> customizers) {
		this.customizers = (customizers != null) ? new ArrayList<>(customizers) : Collections.emptyList();
	}
	@SuppressWarnings("unchecked")
	public <T extends CacheManager> T customize(T cacheManager) {
		LambdaSafe.callbacks(CacheManagerCustomizer.class, this.customizers, cacheManager)
			.withLogger(CacheManagerCustomizers.class)
			.invoke((customizer) -> customizer.customize(cacheManager));
		return cacheManager;
	}
}
```

我以为只要自己实现一个 `CacheManagerCustomizer` 就可以处理 `CaffeineCacheManager` 对象从覆盖 `org.springframework.cache.caffeine.CaffeineCacheManager#cacheBuilder` 字段而开启统计功能，发现它是有这么一个方法，但是它是私有方法无法外部调用

```java CaffeineCacheManager 源码
private void doSetCaffeine(Caffeine<Object, Object> cacheBuilder) {
    if (!ObjectUtils.nullSafeEquals(this.cacheBuilder, cacheBuilder)) {
        this.cacheBuilder = cacheBuilder;
        refreshCommonCaches();
    }
}
```

然后再发现 `CaffeineCacheManager` 有如下三个方法调用了这个私有方法：

```java CaffeineCacheManager 源码
public void setCaffeine(Caffeine<Object, Object> caffeine) {
    Assert.notNull(caffeine, "Caffeine must not be null");
    doSetCaffeine(caffeine);
}
public void setCaffeineSpec(CaffeineSpec caffeineSpec) {
    doSetCaffeine(Caffeine.from(caffeineSpec));
}
public void setCacheSpecification(String cacheSpecification) {
    doSetCaffeine(Caffeine.from(cacheSpecification));
}
```

随便选一个方法，用IDEA快捷键 `Ctrl + Alt + F7` 查看调用，最后发现回到了我们的 `CaffeineCacheConfiguration` 配置类里面有如下代码：

```java CaffeineCacheConfiguration 源码
private void setCacheBuilder(CacheProperties cacheProperties, CaffeineSpec caffeineSpec,
        Caffeine<Object, Object> caffeine, CaffeineCacheManager cacheManager) {
    String specification = cacheProperties.getCaffeine().getSpec();
    if (StringUtils.hasText(specification)) {
        cacheManager.setCacheSpecification(specification);
    }
    else if (caffeineSpec != null) {
        cacheManager.setCaffeineSpec(caffeineSpec);
    }
    else if (caffeine != null) {
        cacheManager.setCaffeine(caffeine);
    }
}
```

然后选第一个设置方法 `cacheManager.setCacheSpecification(specification);` 进入到 `CaffeineCacheManager` 中发现

```java CaffeineCacheManager 源码
public void setCacheSpecification(String cacheSpecification) {
    doSetCaffeine(Caffeine.from(cacheSpecification));
}
```

## 3. 解决问题

发现调用了 `Caffeine.from(cacheSpecification)` 这么一个方法，而这个方法里面又调用了 `CaffeineSpec.parse(spec)` 方法，最后点进去 `CaffeineSpec` 类里面一看，发现 `CaffeineSpec` 类顶部的注释有下面这些内容：

![img_2.png](img_2.png)

都是英文看不懂没关系，咱翻译一下

![img_3.png](img_3.png)

然后就发现有这么一行说明 `recordStats : 设置 Caffeine.recordStats` ，而 `com.github.benmanes.caffeine.cache.Caffeine#recordStats()` 方法就是启用统计功能用的方法，那是不是意味着只要在 `String specification = cacheProperties.getCaffeine().getSpec();` 这个配置项里面配置 `recordStats` 就可以开启统计功能了？ 
那就开始尝试修改配置文件，加上 `recordStats` 参数
```yaml
spring:
  cache:
    cache-names: default
    caffeine:
      spec: maximumSize=500,expireAfterWrite=5s,recordStats
    type: caffeine
```

然后重启服务，再次查看我们的前端页面，发现统计功能已经生效了，数据不再是0了

![img_4.png](img_4.png)

因此在不改一行Java代码的情况下启用了缓存的统计功能。

不过我之前参考的文章是 2020 年的，过了 4 年时间，不确定这个配置是何时引入的，所以又去查看了一下 `SpringBoot` 的源码和 `Caffeine` 的源码，原来这个配置从一开始就已经存在了。

`SpringBoot` 的源码，关键代码行 8 年未变更 [CaffeineCacheConfiguration.java#L72](https://github.com/spring-projects/spring-boot/blame/main/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/cache/CaffeineCacheConfiguration.java#L72)

`Caffeine` 的源码，关键说明内容 9 年未变更 [CaffeineSpec.java#L50](https://github.com/ben-manes/caffeine/blame/master/caffeine/src/main/java/com/github/benmanes/caffeine/cache/CaffeineSpec.java#L50)
