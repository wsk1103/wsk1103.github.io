---
title: "Spring Cloud学习笔记6-Zuul集成Turbine"

url: "https://wsk1103.github.io/"

tags:
  - Spring Cloud
  - 学习笔记
---

#### 备注：  
**官网**：[http://spring.io/projects/spring-cloud](https://cloud.spring.io/spring-cloud-netflix/2.0.x/single/spring-cloud-netflix.html#hystrix-fallbacks-for-routes)

**总纲**：[https://cloud.spring.io/spring-cloud-static/Finchley.SR2/single/spring-cloud.html](https://cloud.spring.io/spring-cloud-static/Finchley.SR2/single/spring-cloud.html)

**Java**： 1.8 +

**MAVEN**： 3.5.0 +

**Spring Boot**：2.0.7.RELEASE

**Spring Cloud**：Finchley

说明：When a circuit for a given route in Zuul is tripped you can provide a fallback response by creating a bean of type ZuulFallbackProvider. Within this bean you need to specify the route ID the fallback is for and provide a ClientHttpResponse to return as a fallback. Here is a very simple ZuulFallbackProvider implementation.

If you would like to choose the response based on the cause of the failure use FallbackProvider which will replace ZuulFallbackProvder in future versions.

当Zuul中给定路径的电路跳闸时，您可以通过创建ZuulFallbackProvider类型的bean来提供回退响应。在此bean中，您需要指定回退所针对的路由ID，并提供ClientHttpResponse作为回退返回。这是一个非常简单的ZuulFallbackProvider实现。

如果您想根据失败原因选择响应，请使用FallbackProvider，它将取代未来版本中的ZuulFallbackProvder。（谷歌翻译）

---

本项目地址：[https://github.com/wsk1103/my-spring-cloud](https://github.com/wsk1103/my-spring-cloud)

---

## 熔断zuul
在使用zuul的过程中，有可能zuul因为某些事故导致服务不可用，此时就可以添加hystrix进行熔断。

zuul已经提供了熔断的功能，只需要实现FallbackProvider接口便可。
#### 1. 沿用zuul项目，实现FallbackProvider
新增包provider -> 新增类 

```
package com.wsk.zuul.provider;

import com.netflix.hystrix.exception.HystrixTimeoutException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.netflix.zuul.filters.route.FallbackProvider;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.stereotype.Component;

import java.io.ByteArrayInputStream;
import java.io.InputStream;
import java.util.Arrays;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/12/11  14:26
 */
@Component
@Slf4j
public class MyProvider implements FallbackProvider {
    @Override
    public String getRoute() {
        return "*";
    }

    @Override
    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
        log.warn(String.format("route:%s,exceptionType:%s,stackTrace:%s", route, cause.getClass().getName(), Arrays.toString(cause.getStackTrace())));
        if (cause instanceof HystrixTimeoutException) {
            return response(HttpStatus.GATEWAY_TIMEOUT);
        } else {
            return response(HttpStatus.INTERNAL_SERVER_ERROR);
        }

    }

    private ClientHttpResponse response(final HttpStatus status) {
        return new ClientHttpResponse() {
            @Override
            public HttpStatus getStatusCode() {
                return status;
            }

            @Override
            public int getRawStatusCode() {
                return status.value();
            }

            @Override
            public String getStatusText() {
                return status.getReasonPhrase();
            }

            @Override
            public void close() {
            }

            @Override
            public InputStream getBody() {
                String msg = String.format("{\"code\": 1103,\"message\": \"%s\"}", status.value());
                return new ByteArrayInputStream(msg.getBytes());
            }

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON);
                return headers;
            }
        };
    }
}


```

**FallbackProvider 说明**

|项	|说明|
|---|----|
getRoute()	|该Provider应用的Route ID，例如：testservice，如果设置为 * ，那就对所有路由生效
fallbackResponse(String route, Throwable cause)|	快速回退失败/响应，即处理异常并返回对应输出/响应内容。route：发生异常的RouteID，cause：触发快速回退/失败的异常/错误
ClientHttpResponse	|Spring提供的HttpResponse接口。可以通过实现该接口自定义Http status、body、header



## 集成Turbine

#### 1. 在Turbine项目的application.yml中添加server-zuul

```
turbine:
#  app-config: service-hi,service-hi2
#指定需要监控的servicename，多个service以,间隔
  app-config: service-feign,service-zuul
#指定集群名称，默认为default，当设立了多个集群时，可以在Hystrix指定集群名称来查看监控
  clusterNameExpression: new String("default")
#合并同一个host多个端口的数据
  combine-host-port: true
```

#### 2. 为zuul项目添加包 configuration -> 类 HystrixConfiguration

其实和feign里面的HystrixConfiguration是一样的。
```
package com.wsk.zuul.configuration;

import com.netflix.hystrix.contrib.metrics.eventstream.HystrixMetricsStreamServlet;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author WuShukai
 * @version V1.0
 * @description
 * @date 2018/12/11  11:48
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

#### 3. 依次启动server，client，feign，zuul，turbine这5个项目
访问：http://localhost:8764/hystrix
，然后是输入框输入：http://localhost:8769/actuator/hystrix.stream
，来监控zuul项目。
然后访问：http://localhost:8769/a/hi?name=sky&token=www
，和：http://localhost:8765/hi?name=wsk  

![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20cloud5/2.png)