---
title: 记一次升级SpringCloud Gateway路由转发的问题
author: 鹏徙南冥
top: false
mathjax: false
categories:
  - 微服务
tags:
  - SpringCloud
  - Gateway
date: 2019-09-20 08:51:30
updated: 2019-09-20 08:51:30
img:
summary:
---
## 说明
本文中使用的SpringBoot版本：2.1.7.RELEASE，SpringCloud依赖版本：Greenwich.SR3。

## 问题描述
在配置一个简单的路由转发时，同样的配置，转发到了不同的路径。配置如下：
```yml
spring:
  cloud:
    gateway:
      routes:
        - id: baidu_route
          uri: https://baidu.com:443/
          predicates:
            - Path=/baidu
```
目的是想当访问localhost:8080时，跳转到`https://baidu.com`。但是升级了SpringCloud版本后，实际却是跳转到了`https://baidu.com/baidu`。并且莫名其妙的使用浏览器访问不进入断点，使用Postman发请求确可以进入，很奇怪。

## 问题原因
首先回顾下SpringCloud Gateway的工作原理图：
![SpringCloud Gateway 工作原理](https://s2.ax1x.com/2019/09/20/nObDL4.png)
从图上可以得知，在进入执行过滤器之前，需要做路由匹配，所以理论上不会在执行过滤器之前就已经对路由地址做了修改，处理过程应该发生在Filter Chain中。

由于本文没有采用任何自定义的Filter，因此图中执行的Filter都是GlobalFilters。可以通过Debug或者集成actuator的方式，查看具体执行了哪些Filter：
```json
{
"org.springframework.cloud.gateway.filter.AdaptCachedBodyGlobalFilter@303c55fa":-2147482648,
"org.springframework.cloud.gateway.config.GatewayNoLoadBalancerClientAutoConfiguration$NoLoadBalancerClientFilter@1319bc2a":10100,
"org.springframework.cloud.gateway.filter.RouteToRequestUrlFilter@b0fd744":10000,
"org.springframework.cloud.gateway.filter.NettyRoutingFilter@502a4156":2147483647,
"org.springframework.cloud.gateway.filter.ForwardRoutingFilter@92d1782":2147483647,
"org.springframework.cloud.gateway.filter.WebsocketRoutingFilter@72976b4":2147483646,
"org.springframework.cloud.gateway.filter.RemoveCachedBodyFilter@64e1377c":-2147483648,
"org.springframework.cloud.gateway.filter.NettyWriteResponseFilter@5416f8db":-1,
"org.springframework.cloud.gateway.filter.ForwardPathFilter@6a1ef65c":0,
"org.springframework.cloud.gateway.filter.GatewayMetricsFilter@726934e2":0
}
```
按照执行顺序从后往前打断点发现，对于URI的处理主要发生在RouteToRequestUrlFilter中。对比新旧版本的代码可以发现，最新版的代码中有这样一段代码：
```java
URI mergedUrl = UriComponentsBuilder.fromUri(uri)
		.scheme(routeUri.getScheme()).host(routeUri.getHost())
		.port(routeUri.getPort()).build(encoded).toUri();
exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, mergedUrl);
```
而与之对应的旧版本中代码如下：
```java
URI mergedUrl = UriComponentsBuilder.fromUri(uri)
		.uri(routeUri);
exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, mergedUrl);
```
其中参数uri为配置文件中predicates的Path路径，routeUri为配置文件中的uri参数。由此可见，在旧版本代码中，重定向的地址即为配置文件的uri，而在新版本中，将Path与uri进行了整合。至此，真相大白。

## 修改处理
如果在实际的情况中，确实有这么一种情况：重定向的URI与原始访问的URI路由不一致，该怎么解决呢？
首先想到的是查看是否能够排除部分默认执行的GlobalFilter，很遗憾我并没有找到相关配置。并且查看Bean加载过程可以发现，只有部分GlobalFilter与依赖类有关，大部分都是不依赖于任何配置加载的。此部分源码位于`GatewayAutoConfiguration`类中，有兴趣的同学可以自行查看。

既然无法排除，那第二个办法就是采用自定义的Filter进行处理。在本例中，可以在`RouteToRequestUrlFilter`执行之前手动将Path置空或者在其之后重新处理URI，关键代码分别如下：
```java
// 置空Path
log.trace("MyFilter start");
exchange = exchange.mutate()
        .request(exchange.getRequest().mutate().path("/").build())
        .build();
return chain.filter(exchange);
```

```java
// 重新处理URI
Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
if (route == null) {
    return chain.filter(exchange);
}
log.trace("MyFilter start");
URI uri = exchange.getRequest().getURI();
boolean encoded = containsEncodedParts(uri);
URI routeUri = route.getUri();
URI mergedUrl = UriComponentsBuilder.fromUri(routeUri).build(encoded).toUri();
exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, mergedUrl);
return chain.filter(exchange);
```
当然针对具体情况具体分析，这里只是提供一种思路。或许还有其他的方式可以做到，这里我也没有读完官方的全部文档（看得脑壳痛）。