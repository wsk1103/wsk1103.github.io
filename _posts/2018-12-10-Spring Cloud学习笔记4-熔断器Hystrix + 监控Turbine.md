---
title: "Spring Cloud学习笔记4-熔断器Hystrix + 监控Turbine"

url: "https://wsk1103.github.io/"

tags:
  - Spring Cloud
  - 学习笔记
---

#### 备注：  
官网：[http://spring.io/projects/spring-cloud](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.1.0.M3/single/spring-cloud-netflix.html#_circuit_breaker_hystrix_clients)

**JAVA**： 1.8 +

**MAVEN**： 3.5.0 +

**Spring Boot**：2.0.7.RELEASE

**Spring Cloud**：Finchley

说明：Netflix has created a library called Hystrix that implements the circuit breaker pattern. In a microservice architecture, it is common to have multiple layers of service calls, as shown in the following example:

Netflix创建了一个名为Hystrix的库，用于实现断路器模式。在微服务架构中，通常有多层服务调用，如以下示例所示：（熔断器监控界面）
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud4/1.png)

A service failure in the lower level of services can cause cascading failure all the way up to the user. When calls to a particular service exceed circuitBreaker.requestVolumeThreshold (default: 20 requests) and the failure percentage is greater than circuitBreaker.errorThresholdPercentage (default: >50%) in a rolling window defined by metrics.rollingStats.timeInMilliseconds (default: 10 seconds), the circuit opens and the call is not made. In cases of error and an open circuit, a fallback can be provided by the developer.  
一句话，Hystrix的快速失败可以有效防止雪崩效应。
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud4/2.png)
---

本项目地址：[https://github.com/wsk1103/my-spring-cloud](https://github.com/wsk1103/my-spring-cloud)

---

## Feign集成Hystrix
延用学习笔记3的项目，对其进行修改
#### 1. 修改pom.xml，增加

```
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
```

#### 2. application.yml 增加新的配置

```
# 因为feign默认集成了hystrix，但是是处于关闭状态。
feign:
  hystrix:
    enabled: true
```

#### 3. 修改SericeFeignApplication

注解增加  
//启动hystrix  
@EnableHystrix  
@EnableCircuitBreaker  

#### 4. 修改 包service -> HiService

在注解@FeignClient新增属性fallback，其中的值HiServiceHystricImpl.class为该接口的实现，在接口访问失败的时候，就会快速调用HiServiceHystricImpl.class里面对应的方法。

```
package com.wsk.feign.service;

import com.wsk.feign.service.impl.HiServiceHystricImpl;
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
@FeignClient(value = "service-client",fallback = HiServiceHystricImpl.class)
public interface HiService {
    @RequestMapping(value = "/hi",method = RequestMethod.GET)
    String say(@RequestParam(value = "name") String name);
}

```

#### 5. 增加接口的的实现类

```
package com.wsk.feign.service.impl;

import com.wsk.feign.service.HiService;
import org.springframework.stereotype.Component;

/**
 * @author WuShukai
 * @version V1.0
 * @description 熔断器，快速失败
 * @date 2018/11/23  14:29
 */
@Component
public class HiServiceHystricImpl implements HiService {

    /**
     * 当访问失败的时候，会快速调用该方法直接返回。
     *
     * @param name
     * @return
     */
    @Override
    public String say(String name) {
        return "sorry,error:" + name;
    }
}

```

#### 6. 修改SericeFeignApplication

新增注解  
//表明开启熔断器  
@EnableHystrix  
@EnableCircuitBreaker  

```
package com.wsk.feign;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
@EnableFeignClients
@RestController
@EnableHystrix
@EnableCircuitBreaker
public class SericeFeignApplication {

    public static void main(String[] args) {
        SpringApplication.run(SericeFeignApplication.class, args);
    }

}

```


## Feign整合Hystrix Dashboard监控数据聚合（Turbine）
Turbine是Netflix开源的将Server-Sent Event（SSE）的JSON数据流聚合成单个流的工具。我们可以通过Turbine将Hystrix生产的监控数据（JSON）合并到一个流中，方便我们对存在多个实例的应用进行监控。


#### 1. 为Feign项目修改application.yml

```
# 熔断器
management:
  endpoints:
    web:
      exposure:
        include: "*"
      cors:
        allowed-origins: "*"
        allowed-methods: "*"
```


#### 2. 为Feign项目新增包configuration -> 新增类 HystrixConfiguration

```
package com.wsk.feign.configuration;

import com.netflix.hystrix.contrib.metrics.eventstream.HystrixMetricsStreamServlet;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/12/10  10:19
 */
@Configuration
public class HystrixConfiguration {

    @Bean(name = "hystrixRegistrationBean")
    public ServletRegistrationBean servletRegistrationBean() {
        ServletRegistrationBean registration = new ServletRegistrationBean(
                new HystrixMetricsStreamServlet(), "/hystrix.stream");
        registration.setName("hystrixServlet");
        registration.setLoadOnStartup(1);
        return registration;
    }

    @Bean(name = "hystrixForTurbineRegistrationBean")
    public ServletRegistrationBean servletTurbineRegistrationBean() {
        ServletRegistrationBean registration = new ServletRegistrationBean(
                new HystrixMetricsStreamServlet(), "/actuator/hystrix.stream");
        registration.setName("hystrixForTurbineServlet");
        registration.setLoadOnStartup(1);
        return registration;
    }
}

```

#### 3. 新建项目turbine

搭建方法同学习笔记1中新建module


#### 4. 修改pom

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.wsk</groupId>
    <artifactId>turbine</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>turbine</name>
    <description>service-turbine</description>

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
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
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

#### 5. 重命名application.properties为application.yml 

```
server:
  port: 8764

spring:
  application:
    name: service-turbine

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

turbine:
#指定需要监控的servicename，多个service以,间隔
  app-config: service-feign
#指定集群名称，默认为default，当设立了多个集群时，可以在Hystrix指定集群名称来查看监控
  clusterNameExpression: new String("default")
#合并同一个host多个端口的数据
  combine-host-port: true

```

#### 6. 修改SericeFeignApplication

新增注解  
@EnableDiscoveryClient  
//开启界面报表  
@EnableHystrixDashboard  
//开启Turbine  
@EnableTurbine   
```
package com.wsk.turbine;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
import org.springframework.cloud.netflix.turbine.EnableTurbine;

@SpringBootApplication
@EnableDiscoveryClient
//开启界面报表
@EnableHystrixDashboard
//开启Turbine
@EnableTurbine
public class TurbineApplication {

    public static void main(String[] args) {
        SpringApplication.run(TurbineApplication.class, args);
    }
}

```

## 输出
#### 1. 依次启动server，client，feign，turbine这4个项目。

![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud4/5.png)
#### 2. 访问turbine项目，http://localhost:8764/hystrix

输入 http://localhost:8765/actuator/hystrix.stream
，点击monitor stream  
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud4/6.png)
#### 3. 可以看到监控界面，该界面处于loading状态，是因为feign没有访问相应的接口。

访问feign相应的接口：http://localhost:8765/hi?name=wsk
，可以看到相应的监控说明。
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud4/9.png)

*注*：一个接口一个图像

#### 4. 监控说明

![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud4/10.png)