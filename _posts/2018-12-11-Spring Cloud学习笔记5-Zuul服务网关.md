---
title: "Spring Cloud学习笔记5-Zuul服务网关"

url: "https://wsk1103.github.io/"

tags:
  - Spring Cloud
  - 学习笔记
---

#### 备注：  
**官网**：[http://spring.io/projects/spring-cloud](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.0.2.RELEASE/single/spring-cloud-netflix.html#_router_and_filter_zuul)

**总纲**：[https://cloud.spring.io/spring-cloud-static/Finchley.SR2/single/spring-cloud.html](https://cloud.spring.io/spring-cloud-static/Finchley.SR2/single/spring-cloud.html)

**Java**： 1.8 +

**MAVEN**： 3.5.0 +

**Spring Boot**：2.0.7.RELEASE

**Spring Cloud**：Finchley

说明：Routing is an integral part of a microservice architecture. For example, / may be mapped to your web application, /api/users is mapped to the user service and /api/shop is mapped to the shop service. Zuul is a JVM-based router and server-side load balancer from Netflix.

路由是微服务架构不可或缺的一部分。例如，/可以映射到您的Web应用程序，/api/users映射到用户服务，/api/shop映射到商店服务。Zuul是Netflix的基于JVM的路由器和服务器端负载均衡器。（谷歌翻译面）

Netflix uses Zuul for the following（有以下作用）:
- Authentication（权限验证）
- Insights
- Stress Testing（压力测试）
- Canary Testing（灰度测试-金丝雀测试）
- Dynamic Routing（动态路由）
- Service Migration（服务迁移）
- Load Shedding（负载均衡）
- Security（安全）
- Static Response handling（处理静态响应)
- Active/Active traffic management（流量管理）

---

本项目地址：[https://github.com/wsk1103/my-spring-cloud](https://github.com/wsk1103/my-spring-cloud)

---


## 搭建服务zuul

搭建方法同学习笔记1中新建module

#### 1. 修改pom.xml

主要增加依赖为

```
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
```

具体pom
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.wsk</groupId>
    <artifactId>zuul</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>service-zuul</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>my-spring-cloud</groupId>
        <artifactId>MySpringCloud</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>


</project>

```

#### 2. 重命名application.properties为application.yml 

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
    
# 路由控制，后续详解
zuul:
  routes:
    a:
      path: /a/**
      serviceId: service-client
    b:
      path: /b/**
      serviceId: service-feign

```

#### 3. 修改ServiceZuulApplication

新增注解  
//开启服务网关  
@EnableZuulProxy  
@EnableEurekaClient  
@EnableDiscoveryClient  
```
package com.wsk.zuul;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@SpringBootApplication
@EnableZuulProxy
@EnableEurekaClient
@EnableDiscoveryClient
public class ServiceZuulApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceZuulApplication.class, args);
    }
}

```

#### 4. 依次启动server，client，feign，zuul
访问：http://localhost:8769/a/go?name=sky
，再访问：http://localhost:8762/a/go?name=sky

## 路由配置说明
#### 1. 参数说明
**路由配置基础参数**

|key|value|
|---|---|
zuul.routes.{routename}	|路由名称，自定义，支持小写字母、-
zuul.routes.{routename}.path	|要路由的路径，支持通配符：?、 、*
zuul.routes.{routename}.serviceId	|注册在Eureka的ServiceName
zuul.routes.{routename}.url	|如果应用没有注册在Eureka，也可以通过指定Url来路由
zuul.ignored-services	|忽略指定的服务,可以配置多个，以,间隔
zuul.ignored-patterns	|忽略指定的路径，可以配置多个，以,间隔。同样支持通配符

**path通配符说明**

|key|value|举例|说明
|---|---|---|---|
?	|匹配单个任意字符	|/sk/?	|/sk/a、/sk/b
*	|匹配任意字符	|sk/*	|/sk/a、/sk/b、/sk/ab
**	|匹配任意字符且支持多级目录	|sk/**	|/sk/a、/sk/b、/sk/ab、/sk/ab/c

#### 实例说明
1. 路由到注册到Eureka的服务


```
zuul:
  routes:
    client:
      path: /client/**
      serviceId: service-client
```
**说明：** 通过路径来指定服务的地址。一般用于拆分服务，也就是微服务。

该项目的配置说明

```
zuul:
  routes:
    a:
      path: /a/**
      serviceId: service-client
    b:
      path: /b/**
      serviceId: service-feign
    c:
      path: /**
      serviceId: service-feign
```
其中path中，将访问/a/**的所有请求转发到service-client相应的接口，例如访问http://localhost:8769/a/go?name=sky，相对于访问service-client服务的go接口。
/b/**同理。

*PS*：路由规则匹配顺序是按配置顺序来的，所以未拆出去的配置在最后。
2. 路由到指定站点

当服务没有注册到注册中心的时候，zuul是无法发现该服务的，但是这个时候我们又需要访问其他网站等等。

```
zuul:
  routes:
    sky:
      path: /sky/**
      url: https://wsk1103.github.io/
```

3. 忽略指定路径

通常适用于某些通用的接口不暴露给外部。这个时候外部就无法访问/**/a和/**/b下的路径。

```
zuul:
  ignored-patterns: /**/a,/**/b
```

4. 忽略指定服务

```
zuul:
  ignored-services: aservice,bservice
```
 
## zuul过滤器
zuul不仅只是路由，并且还能过滤，做一些安全验证
#### 1. 新增包filter -> 新增类 MyFilter
该类继承ZuulFilter，并注解@Component

该过滤器的作用是访问的时候必须带上token，否则无法访问。
```
package com.wsk.zuul.filter;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/11/27  9:50
 */
@Component
@Slf4j
public class MyFilter extends ZuulFilter {

    /**
     * filterType：返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型，具体如下：
     *  pre：路由之前
     *  routing：路由之时
     *  post： 路由之后
     *  error：发送错误调用
     * filterOrder：过滤的顺序
     * shouldFilter：这里可以写逻辑判断，是否要过滤，本文true,永远过滤。
     * run：过滤器的具体逻辑。可用很复杂，包括查sql，nosql去判断该请求到底有没有权限访问。
     */

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        log.info(String.format("%s >>> %s", request.getMethod(), request.getRequestURL().toString()));
        Object accessToken = request.getParameter("token");
        if (accessToken == null) {
            log.warn("token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            try {
                ctx.getResponse().getWriter().write("{\"msg\": \"token is empty\"}");
            } catch (Exception e) {
                log.error(e.getMessage(), e);
            }

            return null;
        }
        log.info("ok");
        return null;
    }
}
```
#### 2. ZuulFilter使用说明
**ZuulFilter方法说明**

|方法名	|说明|
|---|---|
filterType()	|过滤器类型：pre、routing、post、error
filterOrder	|过滤器顺序，用于指定过滤器执行顺序
shouldFilter	|是否要过滤，可以根据当前请求信息判断是否过滤，也可以默认返回true
run	|过滤器执行逻辑，执行具体的过滤操作。

**FilterType说明**

|type|	说明|
|---|---|
pre	|在路由之前执行过滤
routing	|在路由时执行过滤
post	|在路由之后执行过滤
error	|在发生错误时执行过滤


### 参考
1. [https://ken.io/note/spring-cloud-zuul-quickstart](https://ken.io/note/spring-cloud-zuul-quickstart)
2. [https://blog.csdn.net/forezp/article/details/81041012](https://blog.csdn.net/forezp/article/details/81041012)
3. [https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.1.0.M3/single/spring-cloud-netflix.html#_router_and_filter_zuul](https://cloud.spring.io/spring-cloud-netflix/2.0.x/single/spring-cloud-netflix.html#_router_and_filter_zuul)
