---
title: "Spring Cloud学习笔记8-spring cloud gateway(第二代网关)"

url: "https://wsk1103.github.io/"

tags:
  - Spring Cloud
  - 学习笔记
---

#### 备注：  
官网：[http://spring.io/projects/spring-cloud](https://cloud.spring.io/spring-cloud-gateway/2.0.x/single/spring-cloud-gateway.html)

**JAVA**： 1.8 +

**MAVEN**： 3.5.0 +

**Spring Boot**：2.0.7.RELEASE

**Spring Cloud**：Finchley

说明：  
This project provides an API Gateway built on top of the Spring Ecosystem, including: Spring 5, Spring Boot 2 and Project Reactor. Spring Cloud Gateway aims to provide a simple, yet effective way to route to APIs and provide cross cutting concerns to them such as: security, monitoring/metrics, and resiliency.

Spring Cloud Gateway是Spring官方基于Spring 5.0，Spring Boot 2.0和Project Reactor等技术开发的网关，Spring云网关旨在提供一种简单而有效的路由API的方法。Spring Cloud Gateway作为Spring Cloud生态系中的网关，目标是替代Netflix zuul，其不仅提供统一的路由方式，并且基于Filter链的方式提供了网关基本的功能，例如：安全，监控/埋点，和限流等。

**Glossary** 名词解析

- Route（路由）: Route the basic building block of the gateway. It is defined by an ID, a destination URI, a collection of predicates and a collection of filters. A route is matched if aggregate predicate is true. （这是网关的基本构建块。它由一个 ID，一个目标 URI，一组断言和一组过滤器定义。如果断言为真，则路由匹配。）
- Predicate（断言）: This is a Java 8 Function Predicate. The input type is a Spring Framework ServerWebExchange. This allows developers to match on anything from the HTTP request, such as headers or parameters.（这是一个 Java 8 的 Predicate。输入类型是一个 ServerWebExchange。我们可以使用它来匹配来自 HTTP 请求的任何内容，例如 headers 或参数。）
- Filter（过滤器）: These are instances Spring Framework GatewayFilter constructed in with a specific factory. Here, requests and responses can be modified before or after sending the downstream request.（这是org.springframework.cloud.gateway.filter.GatewayFilter的实例，我们可以使用它修改请求和响应。）
 
**how it work**

![image](https://raw.githubusercontent.com/spring-cloud/spring-cloud-gateway/2.0.x/docs/src/main/asciidoc/images/spring_cloud_gateway_diagram.png)

Clients make requests to Spring Cloud Gateway. If the Gateway Handler Mapping determines that a request matches a Route, it is sent to the Gateway Web Handler. This handler runs sends the request through a filter chain that is specific to the request. The reason the filters are divided by the dotted line, is that filters may execute logic before the proxy request is sent or after. All "pre" filter logic is executed, then the proxy request is made. After the proxy request is made, the "post" filter logic is executed.

客户端向Spring Cloud Gateway发出请求。如果网关处理程序映射确定请求与路由匹配，则将其发送到网关Web处理程序。此处理程序运行通过特定于请求的过滤器链发送请求。过滤器被虚线划分的原因是过滤器可以在发送代理请求之前或之后执行逻辑。执行所有“pre”过滤器逻辑，然后进行代理请求。在发出代理请求之后，执行“post”过滤器逻辑。

**VS Netflix Zuul**

Zuul 基于 Servlet 2.5（使用 3.x），使用阻塞 API，它不支持任何长连接，如 WebSockets。而 Spring Cloud Gateway 建立在 Spring Framework 5，Project Reactor 和 Spring Boot 2 之上，使用非阻塞 API，支持 WebSockets，并且由于它与 Spring 紧密集成，所以将会是一个更好的开发体验。

要说缺点，其实 Spring Cloud Gateway 还是有的。目前它的文档还不是很完善，官方文档有许多还处于 TODO 状态，网络上关于它的文章也还比较少。如果你决定要使用它，那么你必须得有耐心通过自己阅读源码来解决可能遇到的问题。（2018.5.7）

[引用：Spring Cloud（十三）：Spring Cloud Gateway（路由）](https://windmt.com/2018/05/07/spring-cloud-13-spring-cloud-gateway-router/)

---

本项目地址：[https://github.com/wsk1103/my-spring-cloud](https://github.com/wsk1103/my-spring-cloud)

---

## 创建项目 service-gateway
搭建方法同学习笔记1中新建module

#### 1. 修改pom.xml

主要增加依赖为

```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
```

具体pom

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.wsk</groupId>
    <artifactId>gateway</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>service-gateway</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>my-spring-cloud</groupId>
        <artifactId>MySpringCloud</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
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
server:
  port: 8766

spring:
  application:
    name: service-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true #在eureka中，服务是以大写的形式注册的，可以转化成小写
      routes:
        - id: service-client #服务唯一ID标识
          uri: lb://service-client # 注册中心的服务id
          predicates:
            - Path=/client/** #请求转发
          filters:
            - StripPrefix=1 #切割请求，去除/client/

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

```

#### 3. 修改ServiceZuulApplication
新增注解 **@EnableEurekaClient**

```
package com.wsk.gateway;

import com.wsk.gateway.resolver.RemoteAddrKeyResolver;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
@EnableEurekaClient
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}

```

#### 4. 依次启动server，client，gateway

访问：http://localhost:8766/client/hi?name=sky 
此时可以看到页面跳转到服务 service-client 的相应接口。

![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/5.png)
## 集成过滤器

### A. 全局过滤器：访问链接必须携带token
---
#### 1. 新建包 filter -> 新建类 TokenFilter 实现 GlobalFilter（定义全局过滤器）, Ordered（定义该过滤器的优先级，值越大则优先级越低）

```
package com.wsk.gateway.filter;

import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.List;

/**
 * @author WuShukai
 * @version V1.0
 * @description 全局过滤器，每次的请求都需要带上token
 * @date 2018/12/12  16:55
 */
@Slf4j
@Data
public class TokenFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String uri = exchange.getRequest().getPath().pathWithinApplication().value();
        log.info("访问的url为：{}", uri);
        String token = exchange.getRequest().getQueryParams().getFirst("token");
        if (token == null || token.isEmpty()) {
            exchange.getResponse().setStatusCode(HttpStatus.BAD_REQUEST);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }


    @Override
    public int getOrder() {
        return -100;
    }
}

```

#### 2. 新建包 config -> 新建类 MyConfig，用于初始化bean

将全局过滤器初始化

```
package com.wsk.gateway.config;

import com.wsk.gateway.filter.TokenFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/12/12  16:56
 */
@Configuration
public class MyConfig {
    @Bean
    public TokenFilter tokenFilter() {
        return new TokenFilter();
    }

}

```
#### 3. 重新启动该gateway服务
访问：http://localhost:8766/client/hi?name=sky ，此时，由于没有带上token，所以后台会直接报错 **BAD_REQUEST** （该值可以在token过滤器里面配置）。  
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/4.png)

修改地址：http://localhost:8766/client/hi?name=sky&token=1103，重新访问  ->   
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/5.png)

### B. 过滤不需要带token也能访问的url
---
#### 新建 gateway-filter-uri.properties
用于排除不需要带token的url

```
# 过滤不需要token的uri，多个以,分开。
gateway.filter.uri=/login,/logout
```
#### 2. 在config包新建类 PropertiesConfig （读取自定义的properties）

```
package com.wsk.gateway.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;

import java.util.Arrays;
import java.util.List;

/**
 * @author WuShukai
 * @version V1.0
 * @description 读取自定义的properties
 * @date 2018/12/14  11:17
 */

@Configuration
//配置前缀
@ConfigurationProperties(prefix = "gateway.filter")
//配置文件名
@PropertySource("classpath:/gateway-filter-uri.properties")
@Data
public class PropertiesConfig {

    //前缀名（gateway.filter） + 后缀名（uri）
    private String uri;

    //将定义的uri转化成list
    public List<String> handleUri() {
        return Arrays.asList(uri.split(","));
    }
}

```
#### 3. 修改 TokenFilter

主要为新增代码片

```
        for (String url : all) {
            //过滤不需要拥有token的连接
            if (uri.startsWith(url)) {
                log.info("不需要拥有token的uri：{}", uri);
                return chain.filter(exchange);
            }
        }
```

新的类
```
package com.wsk.gateway.filter;

import com.wsk.gateway.config.PropertiesConfig;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.List;

/**
 * @author WuShukai
 * @version V1.0
 * @description 全局过滤器，每次的请求都需要带上token
 * @date 2018/12/12  16:55
 */
@Slf4j
@Data
@EnableConfigurationProperties(PropertiesConfig.class)
public class TokenFilter implements GlobalFilter, Ordered {

    @Autowired
    private PropertiesConfig propertiesConfig;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        List<String> all = propertiesConfig.handleUri();
        String uri = exchange.getRequest().getPath().pathWithinApplication().value();
        log.info("访问的url为：{}", uri);
        for (String url : all) {
            //过滤不需要拥有token的连接
            if (uri.startsWith(url)) {
                log.info("不需要拥有token的uri：{}", uri);
                return chain.filter(exchange);
            }
        }
        String token = exchange.getRequest().getQueryParams().getFirst("token");
        if (token == null || token.isEmpty()) {
            exchange.getResponse().setStatusCode(HttpStatus.BAD_REQUEST);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }


    @Override
    public int getOrder() {
        return -100;
    }
}

```


### C. 局部过滤器：控制台显示访问url和参数
---
#### 1. 在包filter新建类 SkyGatewayFilterFactory 继承 AbstractGatewayFilterFactory<SkyGatewayFilterFactory.Config>
其中SkyGatewayFilterFactory.Config为静态内部类，具体可以参考 **HystrixGatewayFilterFactory** (自带的全局熔断器)

```
package com.wsk.gateway.filter;

import lombok.Data;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.factory.AbstractGatewayFilterFactory;
import reactor.core.publisher.Mono;

import java.util.Collections;
import java.util.List;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/12/12  17:01
 */
@Slf4j
public class SkyGatewayFilterFactory extends AbstractGatewayFilterFactory<SkyGatewayFilterFactory.Config> {

    private static final String TIME_BEGIN = "TimeBegin";
    private static final String KEY = "withParams";

    @Override
    public List<String> shortcutFieldOrder() {
        return Collections.singletonList(KEY);
    }

    public SkyGatewayFilterFactory() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            exchange.getAttributes().put(TIME_BEGIN, System.currentTimeMillis());
            return chain.filter(exchange).then(
                    Mono.fromRunnable(() -> {
                        Long startTime = exchange.getAttribute(TIME_BEGIN);
                        if (startTime != null) {
                            StringBuilder sb = new StringBuilder(exchange.getRequest().getURI().getRawPath())
                                    .append(": ")
                                    .append(System.currentTimeMillis() - startTime)
                                    .append("ms");
                            if (config.isWithParams()) {
                                sb.append(" params:").append(exchange.getRequest().getQueryParams());
                            }
                            log.info(sb.toString());
                        }
                    })
            );
        };
    }

    @Data
    static class Config {
        private boolean withParams;
    }

}

```
#### 2. 修改application.yml

主要在filters 里面增加 *- Sky=true* ，其中*Sky* 为过滤器的前缀，后缀为*GatewayFilterFactory* ，gateway是根据这个前缀去加载相应的 *GatewayFilterFactory*，
```
server:
  port: 8766

spring:
  application:
    name: service-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true #在eureka中，服务是以大写的形式注册的，可以转化成小写
      routes:
        - id: service-client #服务唯一ID标识
          uri: lb://service-client # 注册中心的服务id
          predicates:
            - Path=/client/** #请求转发
          filters:
            - Sky=true
            - StripPrefix=1 #切割请求，去除/client/
```
#### 3. 在包config 声明一个新的bean
初始化局部过滤器。
```
    @Bean
    public SkyGatewayFilterFactory skyGatewayFilterFactory() {
        return new SkyGatewayFilterFactory();
    }
```

#### 4. 重新启动gateway
访问：http://localhost:8766/client/hi?name=sky&token=1103  
可以在控制台看到  
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/6.png)

### D. 局部过滤器-基于令牌桶的限流过滤器
---
#### 1. 在包filter新建类 SkyRateLimitByIpGatewayFilterFactory 继承 AbstractNameValueGatewayFilterFactory
**AbstractNameValueGatewayFilterFactory** 是spring cloud gateway中定义的一个抽象类，其中的静态类config有name，value这2个属性，本次使用这2个属性来判断是非开启该过滤器。

```
package com.wsk.gateway.filter;

import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Bucket4j;
import io.github.bucket4j.Refill;
import lombok.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GatewayFilter;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.factory.AbstractNameValueGatewayFilterFactory;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * @author WuShukai
 * @version V1.0
 * @description 基于令牌桶的限流过滤器，必须继承AbstractGatewayFilterFactory或者实现GatewayFilterFactory
 * @date 2018/12/12  17:12
 */
@Builder
@Data
@ToString
@EqualsAndHashCode(callSuper = false)
@AllArgsConstructor
@NoArgsConstructor
@Slf4j
public class SkyRateLimitByIpGatewayFilterFactory extends AbstractNameValueGatewayFilterFactory implements GatewayFilter, Ordered {

    /**
     * 桶的最大容量，即能装载 Token 的最大数量
     */
    private int capacity;

    /**
     * 每次 Token 补充量
     */
    private int refillTokens;

    /**
     * 补充 Token 的时间间隔
     */
    private Duration refillDuration;

    private static final Map<String, Bucket> CACHE = new ConcurrentHashMap<>();

    private Bucket createNewBucket() {
        Refill refill = Refill.greedy(refillTokens, refillDuration);
        Bandwidth limit = Bandwidth.classic(capacity, refill);
        return Bucket4j.builder().addLimit(limit).build();
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String ip = exchange.getRequest().getRemoteAddress().getAddress().getHostAddress();
        Bucket bucket = CACHE.computeIfAbsent(ip, k -> createNewBucket());

        log.info("IP: " + ip + ", TokenBucket Available Tokens: " + bucket.getAvailableTokens());
        if (bucket.tryConsume(1)) {
            return chain.filter(exchange);
        } else {
            //请求太多，服务器之间返回 TOO_MANY_REQUESTS 429
            exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
            return exchange.getResponse().setComplete();
        }
    }

    @Override
    public int getOrder() {
        return -1000;
    }

    @Override
    public GatewayFilter apply(NameValueConfig config) {
        //开启限流
        if ("open".equals(config.getName())) {
            return this;
        }
        return (exchange, chain) -> chain.filter(exchange);
    }
}

```
#### 2. 修改类MyConfig
新增**bean** **SkyRateLimitByIpGatewayFilterFactory**
```
    @Bean
    public SkyRateLimitByIpGatewayFilterFactory skyRateLimitByIpGatewayFilterFactory() {
        return new SkyRateLimitByIpGatewayFilterFactory(10, 1, Duration.ofSeconds(1));
    }
```
参数分析：

```
    /**
     * 桶的最大容量，即能装载 Token 的最大数量-----》10 个
     */
    private int capacity;
    /**
     * 每次 Token 补充量  -------》 1个
     */
    private int refillTokens;

    /**
     * 补充 Token 的时间间隔 -------》 1秒
     */
    private Duration refillDuration;
```

#### 3. 修改application.yml

主要为新增了过滤器**SkyRateLimitByIp**配置 和增加**Redis**配置（因为该限流需要使用到Redis）。

```
spring:
  application:
    name: service-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true #在eureka中，服务是以大写的形式注册的，可以转化成小写
      routes:
        - id: service-client #服务唯一ID标识
          uri: lb://service-client # 注册中心的服务id
          predicates:
            - Path=/client/** #请求转发
          filters:
            - StripPrefix=1 #切割请求，去除/client/
            - name: SkyRateLimitByIp
              args:
                name: open
                value: close
            - name: Retry #重试机制
              args:
                retries: 3
                statuses: BAD_GATEWAY

# gateway限流工具
  redis:
    host: localhost
    port: 6379
```
#### 4. 修改pom.xml
增加Redis依赖和bucket4j依赖

```
        <!--基于令牌桶的限流过滤器-->
        <dependency>
            <groupId>com.github.vladimir-bukhtoyarov</groupId>
            <artifactId>bucket4j-core</artifactId>
            <version>4.2.0</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
        </dependency>
```
#### 5. 重启gateway项目

访问：http://localhost:8766/client/go?name=ss&token=11

可以看到控制台输出：  
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/8.png)

短时间重复刷新页面：  
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/9.png)

可以看到服务器响应 **TOO_MANY_REQUESTS** 429

查看控制台，可以看到令牌在减少，并且每1秒有回复1个的迹象，当令牌减少到O的时候，继续访问就会出现 **TOO_MANY_REQUESTS**  
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/10.png)

### E. 基于Java链的局部过滤器的声明-基于令牌桶的限流过滤器

上面的几种过滤器都是基于yml，这次用Java来声明一个过滤器。

**SkyRateLimitByIpGatewayFilterFactory** 必须实现接口**GatewayFilter** , **Ordered**

自定义的Java链的过滤器比较容易，只需要在启动类**GatewayApplication**中添加**bean**，如何new相应的构造链。

```
    @Bean
    public RouteLocator customerRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route(r -> r.path("/client/**")
                        .filters(f -> f.stripPrefix(1)
                                .filter(new SkyRateLimitByIpGatewayFilterFactory(10, 1, Duration.ofSeconds(1)))
                                //多个过滤器的时候，可以继续构造下去
                        )
                        .uri("lb://service-client")
                        .order(0)
                        .id("service-client")
                )
                .build();
    }
```

这样的声明可以和yml配置混合着用，过滤器的优先级主要看该过滤器实现 Ordered 中**getOrder()** 方法返回的数字，数字越小，表示优先级越高。

### F. 全局过滤器-全局熔断

#### 1. 修改pom.xml
新增**hystrix** 依赖

```
        <!--全局熔断-->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
```

#### 2. application.yml配置hystrix

配置**hystrix** 和 熔断时间 **timeoutInMilliseconds**
```
spring:
  application:
    name: service-gateway
  cloud:
    gateway:
      default-filters:
        - Sky=true
        - name: Hystrix #全局异常熔断处理，必须为Hystrix，会自动匹配HystrixGatewayFilterFactory
          args:
            name: fallbackcmd #必须为fallbackcmd，用于HystrixGatewayFilterFactory中bean的声明
            fallbackUri: forward:/fallback #当熔断的时候，跳转该链接
            
            
#Hystrix的fallbackcmd的时间，默认为1s
hystrix:
  command:
    fallbackcmd:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 2000 #毫秒
```

#### 3. 新建包 controller -> 新建类HystrixController
声明**fallback** 这个熔断响应

```
package com.wsk.gateway.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/12/13  11:33
 */
@RestController
@Slf4j
public class HystrixController {

    @GetMapping("/fallback")
    public String fallback() {
        return "hystrix fall";
    }

}

```

#### 4. 在项目client中，新增代码片段

Hystrix的**fallbackcmd** 的时间为2秒，所以我们设置一个接口地址，里面sleep了4秒，用来模拟超时

在包 **controller** 的类 **OneController** 新增代码片段

```
    @GetMapping("/sky/histrix")
    public String myHistrix(@RequestParam(value = "name", defaultValue = "true") boolean name) throws InterruptedException {
        if (name) {
            TimeUnit.SECONDS.sleep(4);
        }
        return "success";
    }
```

#### 5. 重启服务client和gateway
访问：http://localhost:8766/client/sky/histrix?token=11

2秒后响应

![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/7.png)

证明全局熔断成功。





