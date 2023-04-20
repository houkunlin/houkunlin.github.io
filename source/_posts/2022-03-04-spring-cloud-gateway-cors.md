---
title: SpringCloudGateway跨域配置
date: 2022-03-04 10:58:47
updated: 2022-03-04 10:58:47
tags:
- SpringBoot
- SpringCloud
---

先放一个可行的配置信息，后面再说过程。

```yaml
spring:
  cloud:
    gateway:
      globalcors:
        # 一个全局默认的跨域配置，但是单独配置这个还是无法解决跨域问题
        cors-configurations:
          '[/**]':
            allowedOriginPatterns: "*"
            allowedHeaders: "*"
            allowedMethods: "*"
            allowCredentials: true
      default-filters:
        # 然后再对跨域Header配置做去重复处理，这样就能够保证跨域信息的完整度
        # 升级 SpringBoot 3.0 之后，发现这个配置得写在前面才行，写在前面的最后执行
        - DedupeResponseHeader=Vary Access-Control-Allow-Origin Access-Control-Allow-Credentials, RETAIN_LAST
        # 为了防止后端未配置跨域导致浏览器提示缺少跨域配置信息而请求失败，因此加上了默认的跨域Header配置
        - AddResponseHeader=Access-Control-Allow-Origin, *
        - AddResponseHeader=Access-Control-Allow-Methods, *
        - AddResponseHeader=Access-Control-Allow-Headers, *
        - AddResponseHeader=Access-Control-Allow-Credentials, true
```



最近在使用 Nginx 转发到 SpringCloudGateway 发现了跨域问题，SpringCloud是已经配置了跨域信息，但是手机端（uni-app）依旧无法请求成功，经过尝试在Nginx配置跨域头部同样不行，最后排查了很久，得出了以下结论：

**一、发送请求时出现问题**

问题描述：SpringCloudGateway对跨域做了校验（OPTIONS请求【后端使用了其他自定义Header，因此需要OPTIONS请求】），导致手机端那边测试中出现的接口403问题

解决方案：SpringCloudGateway配置跨域信息

**二、返回数据出现问题**

问题描述：解决了 OPTIONS 403 问题后 POST 提示跨域问题（经抓包实际后端有登录成功json返回），主要错误有两个，要么是存在多个 Origin 相关数据（MulitCorsOrigin），要么是缺少 Origin 相关数据（MissingCorsOrigin）；

解决方案：增加默认过滤器配置，过滤器先添加跨域头（补全补完整跨域信息）然后再对跨域头去重处理

**问题总结：**

第一个坑：对 SpringCloudGateway 转发特性不是非常了解导致的，以为 SpringCloudGateway 不会处理跨域配置，而是直接转发给后端实例，结果 Gateway 会处理 OPTIONS 跨域检查

第二个坑：对 SpringCloudGateway 配置不够了解导致的，Gateway 配置了跨域后 OPTIONS 200 但是 POST CORS 错误就是 Gateway 和 Nginx 两者配置冲突导致的（Nginx也配了跨域）

**冲突描述：**

一、Nginx 加了跨域，Gateway 不配置跨域，OPTIONS 未请求到后端实例，Gateway 提示跨域错误403

二、Nginx 加了跨域，Gateway 配置跨域，但，OPTIONS 请求到后端实例，导致多个跨域请求头浏览器报错（后端配了跨域参数）

三、Nginx 加了跨域，Gateway 的跨域配置也有一个 Access-Control-Allow-Origin 就导致浏览器报 MulitCorsOrigin 问题

四、Nginx 不加跨域，Gateway 的跨域配置有 Access-Control-Allow-Origin ，但是没有 Access-Control-Allow-Methods 就导致浏览器报 MissingCorsOrigin 问题

