---
title: "Spring Cloud学习笔记3-服务消费者（Feign）"

url: "https://wsk1103.github.io/"

tags:
  - Spring Cloud
  - 学习笔记
---

#### 备注：  
官网：[http://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html](http://cloud.spring.io/spring-cloud-static/Finchley.RELEASE/single/spring-cloud.html)

**JAVA**： 1.8 +

**MAVEN**： 3.5.0 +

**Spring Boot**：2.1.1.RELEASE

**Spring Cloud**：Finchley

说明：Feign is a declarative web service client. It makes writing web service clients easier. To use Feign create an interface and annotate it. It has pluggable annotation support including Feign annotations and JAX-RS annotations. Feign also supports pluggable encoders and decoders. Spring Cloud adds support for Spring MVC annotations and for using the same HttpMessageConverters used by default in Spring Web. Spring Cloud integrates Ribbon and Eureka to provide a load balanced http client when using Feign.

Feign是一个声明性的Web服务客户端。它使编写Web服务客户端变得更容易。要使用Feign，请创建一个界面并对其进行注释。它具有可插入的注释支持，包括Feign注释和JAX-RS注释。Feign还支持可插拔编码器和解码器。Spring Cloud增加了对Spring MVC注释的支持，并使用了Spring Web中默认使用的相同HttpMessageConverters。Spring Cloud集成了Ribbon和Eureka，可在使用Feign时提供负载均衡的http客户端。（谷歌翻译）

---

本项目地址：[https://github.com/wsk1103/my-spring-cloud](https://github.com/wsk1103/my-spring-cloud)

---

## 启动服务
1. 启动学习笔记2中的3个服务，server，client，client2。


## 搭建Feign服务：feign
搭建方法同学习笔记1中新建module
1. 修改pom文件

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.wsk</groupId>
    <artifactId>feign</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>serice-feign</name>
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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
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
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
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
  port: 8765

spring:
  application:
    name: service-feign

```

3. 修改SericeFeignApplication

需要在SericeFeignApplication类上新增注解  
@EnableEurekaClient  
@EnableDiscoveryClient  
//负载均衡  
@EnableFeignClients  
@RestController  



```
package com.wsk.feign;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
@EnableFeignClients
@RestController
public class SericeFeignApplication {

    public static void main(String[] args) {
        SpringApplication.run(SericeFeignApplication.class, args);
    }
}

```


4. 新增包service -> 新增类 HiService

```
package com.wsk.feign.service;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/11/23  11:26
 */
//该value必须与注册中心中相应的服务名一致
@FeignClient(value = "service-client")
public interface HiService {
    //会根据配置跳转请求另外服务的相应接口
    @RequestMapping(value = "/hi",method = RequestMethod.GET)
    String say(@RequestParam(value = "name") String name);
}

```


5. 新增包controller -> 新增类HiController

```
package com.wsk.feign.controller;


import com.wsk.feign.service.HiService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/11/23  11:27
 */
@RestController
public class HiController {

    @Autowired
    private HiService hiService;

    @GetMapping(value = "/hi")
    public String sayHi(@RequestParam String name) {
        return hiService.say( name );
    }

}

```

6. 启动服务SericeFeignApplication

多次访问http://localhost:8765/hi?name=sky,浏览器交替显示：  
```
hi sky,i am from port:8762

hi sky,i am from port:8763
```