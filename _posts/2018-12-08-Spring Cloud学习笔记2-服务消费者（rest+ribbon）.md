---
title: "Spring Cloud学习笔记2-服务消费者（rest+ribbon）"

url: "https://wsk1103.github.io/"

tags:
  - Spring Cloud
  - 学习笔记
---

#### 备注：  
官网：[http://spring.io/projects/spring-cloud](http://spring.io/projects/spring-cloud)

**JAVA**： 1.8 +

**MAVEN**： 3.5.0 +

**Spring Boot**：2.1.1.RELEASE

**Spring Cloud**：Finchley

说明：Ribbon is a client-side load balancer that gives you a lot of control over the behavior of HTTP and TCP clients. Feign already uses Ribbon, so, if you use @FeignClient, this section also applies.

A central concept in Ribbon is that of the named client. Each load balancer is part of an ensemble of components that work together to contact a remote server on demand, and the ensemble has a name that you give it as an application developer (for example, by using the @FeignClient annotation). On demand, Spring Cloud creates a new ensemble as an ApplicationContext for each named client by using RibbonClientConfiguration. This contains (amongst other things) an ILoadBalancer, a RestClient, and a ServerListFilter.

Ribbon是一个客户端负载均衡器，可以让您对HTTP和TCP客户端的行为进行大量控制。Feign已使用Ribbon，因此，如果您使用@FeignClient，此部分也适用。

Ribbon中的一个核心概念是指定客户端的概念。每个负载均衡器都是一组组件的一部分，这些组件一起工作以按需联系远程服务器，并且该集合具有您作为应用程序开发人员提供的名称（例如，通过使用@FeignClient批注）。根据需要，Spring Cloud通过使用RibbonClientConfiguration为每个命名客户端创建一个新的集合作为ApplicationContext。这包含（除其他外）ILoadBalancer，RestClient和ServerListFilter。（谷歌翻译）

---

本项目地址：[https://github.com/wsk1103/my-spring-cloud](https://github.com/wsk1103/my-spring-cloud)

---

在开发过程中，基本上都会根据业务进行相应的拆分成为多个微服务。服务之间的通讯是基于http restful。Spring cloud有两种服务调用方式，一种是ribbon+restTemplate，另一种是feign（集成了ribbon+restTemplate）。

## 新增一个服务client2，用于模拟负载均衡。
copy学习笔记1中的client，重新命名为client2，并且将端口修改8763。依次启动server，client，client2这3个服务。

可以在注册中心看到这2个服务已经注册了。

## 搭建Ribbon服务：ribbon
搭建方法同学习笔记1中新建module
1. 修改pom文件

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.wsk</groupId>
    <artifactId>ribbon</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>service-ribbon</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>my-spring-cloud</groupId>
        <artifactId>MySpringCloud</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <properties>
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
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
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

2. 重命名application.properties为application.yml  

```
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

server:
  port: 8764

spring:
  application:
    name: service-ribbon
```

3. 修改ServiceRibbonApplication

需要在ServiceRibbonApplication类上新增注解 **@EnableEurekaClient**
和
**@EnableDiscoveryClient**
```
package com.wsk.ribbon;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableEurekaClient
//允许发现服务客户端
@EnableDiscoveryClient
public class ServiceRibbonApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceRibbonApplication.class, args);
    }

    @Bean
    @LoadBalanced
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

```

4. 新建包service -> 新建类HelloService

该类需要增加注解 **@Service**
```
package com.wsk.ribbon.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/11/23  10:46
 */
@Service
public class HelloService {

    @Autowired
    RestTemplate restTemplate;

    public String hiService(String name) {
        //微服务调用
        return restTemplate.getForObject("http://SERVICE-CLIENT/hi?name="+name,String.class);
    }
}

```

5. 新建包controller -> 新建类HiController

访问该类中的方法的时候，该类会自动根据负载均衡的规则轮训注册中心的SERVICE-CLIENT。
```
package com.wsk.ribbon.controller;

import com.wsk.ribbon.service.HelloService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/11/23  10:47
 */
@RestController
public class HiController {
    @Autowired
    private HelloService helloService;

    @GetMapping(value = "/hi")
    public String hi(@RequestParam String name) {
        return helloService.hiService( name );
    }
}

```

6. 启动ServiceRibbonApplication，多次访问http://localhost:8764/hi?name=sky
浏览器会交通显示：

```
hi sky,i am from port:8762

hi sky,i am from port:8763
```
