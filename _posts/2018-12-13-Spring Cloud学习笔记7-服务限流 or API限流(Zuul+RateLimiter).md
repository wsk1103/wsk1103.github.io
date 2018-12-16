---
title: "Spring Cloud学习笔记7-服务限流 or API限流(Zuul+RateLimiter)"

url: "https://wsk1103.github.io/"

tags:
  - Spring Cloud
  - 学习笔记
---

#### 备注：  
**官网**：[http://spring.io/projects/spring-cloud](https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit)

**总纲**：[https://cloud.spring.io/spring-cloud-static/Finchley.SR2/single/spring-cloud.html](https://cloud.spring.io/spring-cloud-static/Finchley.SR2/single/spring-cloud.html)

**JAVA**： 1.8 +

**MAVEN**： 3.5.0 +

**Spring Boot**：2.0.7.RELEASE

**Spring Cloud**：Finchley

说明：  
**RateLimiter** ：RateLimiter是Google开源的实现了令牌桶算法的限流工具（速率限制器）。[http://ifeve.com/guava-ratelimiter/](http://ifeve.com/guava-ratelimiter/)

**[Spring Cloud Zuul RateLimiter](https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit)** ：结合了Zuul对RateLimiter进行了封装，通过实现ZuulFilter提供了服务限流功能

|限流粒度/类型	|说明|
|---|---|
Authenticated User|	针对请求的用户进行限流
Request Origin	|针对请求的Origin进行限流
URL	|针对URL/接口进行限流
Service	|针对服务进行限流，如果没有配置限流类型，则此类型生效

---

本项目地址：[https://github.com/wsk1103/my-spring-cloud](https://github.com/wsk1103/my-spring-cloud)

---

## 服务限流
延用zuul项目
#### 1. 修改pom.xml，加入新的依赖

```
<!-- https://mvnrepository.com/artifact/com.marcosbarbero.cloud/spring-cloud-zuul-ratelimit -->
<dependency>
    <groupId>com.marcosbarbero.cloud</groupId>
    <artifactId>spring-cloud-zuul-ratelimit</artifactId>
    <version>2.2.1.RELEASE</version>
</dependency>
//增加redis，配合redis使用
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```
#### 2. 修改application.yml
增加了redis配置和rate limit配置
```
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

server:
  port: 8769

spring:
  application:
    name: service-zuul
  redis:
    host: localhost
    port: 6379

zuul:
  routes:
    a:
      path: /a/**
      serviceId: service-client
    b:
      path: /b/**
      serviceId: service-feign
  ratelimit:
    key-prefix: wsk
    enabled: true
    repository: REDIS
    behind-proxy: true
    default-policy-list: #optional - will apply unless specific policy exists
      - limit: 1 #optional - request number limit per refresh interval window
        quota: 1 #optional - request time limit per refresh interval window (in seconds)
        refresh-interval: 3 #default value (in seconds)
#        type: #optional
#          - user
#          - origin
#          - url
    policy-list:
      a: #需要和服务同名
        - limit: 10 #optional - request number limit per refresh interval window
          quota: 100 #optional - request time limit per refresh interval window (in seconds)
          refresh-interval: 30 #default value (in seconds)
      b:
        - limit: 1 #optional - request number limit per refresh interval window
          quota: 1 #optional - request time limit per refresh interval window (in seconds)
          refresh-interval: 3 #default value (in seconds)
#          type: #optional
#            - user
#            - origin
#            - url
#          type: #optional value for each type
#            - user=anonymous
#            - origin=somemachine.com
#            - url=/api #url prefix
#            - role=user
```
#### 结果实例

![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/3.png)

## 参数说明
Property namespace: zuul.ratelimit

|Property name|	Values	|Default Value|说明
|----|----|----|----|
enabled|true/false|false|是否启用限流
behind-proxy|true/false|false|
key-prefix|String|${spring.application.name:rate-limit-application}|限流key前缀
repository|CONSUL, REDIS, JPA, BUCKET4J_JCACHE, BUCKET4J_HAZELCAST, BUCKET4J_INFINISPAN, BUCKET4J_IGNITE|-|必填，使用redis即可
default-policy-list|List of Policy|-|默认策略
policy-list|Map of Lists of Policy|-|自定义策略
postFilterOrder|int|FilterConstants.SEND_RESPONSE_FILTER_ORDER - 10|postFilter过滤顺序
preFilterOrder|int|FilterConstants.FORM_BODY_WRAPPER_FILTER_ORDER|preFilter过滤顺序

Policy properties:

Property name|	Values|	Default Value|说明
|----|----|----|---|
limit|number of calls|-|单位时间内请求次数限制
quota|time of calls|-|单位时间内累计请求时间限制（秒），非必要参数
refresh-interval|seconds|60|单位时间（秒），默认60秒
type|[ORIGIN, USER, URL, ROLE]|[]|限流方式：ORIGIN, USER, URL（每个Url 在z秒内访问次数不得超过x次且总计访问时间这不得超过y秒）