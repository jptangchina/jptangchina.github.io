---
title: >-
  SpringCloud微服务ClientException:Load balancer does not have available server for
  client异常解决方案
author: 鹏徙南冥
top: false
mathjax: false
categories:
  - 微服务
tags:
  - SpringCloud
  - SpringBoot
  - ribbon
  - feign
date: 2019-08-25 10:07:17
updated: 2019-08-25 10:07:17
img:
summary:
---

## 遇到的问题
最近在使用最新版本的SpringCloud编写demo时发现的问题。使用Feign在进行服务间调用时，会提示异常：
```java
ClientException:Load balancer does not have available server for xxx
```
使用的版本为：
```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.7.RELEASE</version>
    </parent>
	
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.SR2</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
```

提示信息已经很明确了。在早期的F版本左右，搭建一个简单的调用实例，是不需要显示的指明Ribbon配置的，但是在最新的版本中，需要主动指明启用Ribbon或单独进行配置。现将问题记录于此以便查阅。

## 解决方案

### 早期版本可能的方案
如果在某度上搜索这个问题，其中之一的解决方案是，引入
```xml
<dependency>  
    <groupId>org.springframework.cloud</groupId>  
    <artifactId>spring-cloud-starter-eureka</artifactId>  
</dependency>  
```
同样的，这可能是早期版本的解决方案，并不适用现在的版本。事实上，如果去中央仓库上查看这个包，会发现其已经被标记为`deprecated`，并且推荐使用`spring-cloud-starter-netflix-eureka-client`替代。

### 最新版本的方案

#### 显示的声明启用Ribbon
在消费端的配置文件中设置：
```yml
ribbon:
  eureka:
    enabled: true
```
此项设置会自动基于Eureka做负载均衡。

#### 手动配置Ribbon
如果项目中并没有使用Eureka，可以手动进行配置文件配置或者编写代码进行配置。

配置文件配置如下：
```yml
# 生产者服务名
micro-service-producer:
  ribbon:
    # 服务地址
    listOfServers: localhost:8200,localhost:8201
```

代码配置如下：
```java
@RibbonClient(name = "micro-service-producer", configuration = RibbonConfig.class)
public class RibbonConfig {
    private static final String listOfServers = "localhost:8200,localhost:8201";

    @Bean
    public ServerList<Server> ribbonServerList() {
        List<Server> list = Lists.newArrayList();
        if (!Strings.isNullOrEmpty(listOfServers)) {
            for (String s: listOfServers.split(",")) {
                list.add(new Server(s.trim()));
            }
        }
        return new StaticServerList<>(list.toArray(new Server[0]));
    }
}
```

## 总结

上面的问题可能并不是导致这个报错的唯一原因，但应该是最有可能的原因。结合版本、使用的技术再确定解决方案，不可盲目相信网上的答案。