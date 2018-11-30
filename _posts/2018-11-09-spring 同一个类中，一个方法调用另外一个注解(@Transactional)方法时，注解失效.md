---
title: "spring同一个类中，一个方法调用另外一个注解(@Transactional)方法时，注解失效"

url: "https://wsk1103.github.io/"

tags:
  - 错误笔记
---

**基于spring 3.2.9**   
参考 [https://www.ibm.com/developerworks/cn/java/j-master-spring-transactional-use/index.html](https://www.ibm.com/developerworks/cn/java/j-master-spring-transactional-use/index.html)
### 1. Transactional是什么
 Transactional用来声明事务的，包括事务的begin和commit。被注解的public方法或者类在被调用的时候，spring会为该public方法或者类中的所有public方法生成一个代理类来代理被注解的方法。  

例如：  
原类->
```
public class A {
    @Transactional
    public void a() {
        ...            
    }
}
```

被代理后

```
public class Proxy$A {

    A a = new A();

    //spring扫描注解后，为注解的方法插入一个startTransaction()方法。
    public void a () {
        startTransaction();
        a.a();
        commitTransactionAfterReturning();
    }
}
```

## 2. 怎么使用@Transactional  
目前比较流行的使用是基于Java注解声明。
分为2个步骤：  

1. 在spring.xml文件中声明事务的配置信息

```
    <!--======= 事务配置 Begin ================= -->
    <!-- 事务管理器（由Spring管理MyBatis的事务） -->
    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- 关联数据源 -->
        <property name="dataSource" ref="dataSource"></property>
    </bean>
    <!-- 注解事务 -->
    <tx:annotation-driven/>
    <!--======= 事务配置 End =================== -->
```
2. 在public方法或者类上声明@Transactional 

@Transactional 注解属性说明：

| 属性名 | 说明 |
|----|-----|
value | 当在xml配置文件中配置多个TransactionManager的时候，可以指定使用哪个事务管理器
propagation | 事务的传播行为，默认为Propagation.REQUIRED（表示启动事务）。PROPAGATION_SUPPORTS:如果当前存在事务，则加入该事务，如果没有事务，则以非事务的方式继续进行。 PROPAGATION_NOT_SUPPORTED：以非事务的方法运行，如果当前存在事务，则将事务挂起。PROPAGATION_NEVER：以非事务的方法运行，如果当前存在事务，则抛出异常。
isolation | 事务的隔离等级，默认为Isolation.DEFAULT。必须返回 TransactionDefinition 接口上定义的ISOLATION_XXX 常量之一。只有结合PROPAGATION_REQUIRED 或者 PROPAGATION_REQUIRES_NEW 一起声明才有意义。
timeout | 默认值为 -1（不超时），单位秒。表示事务必须在规定的时间内处理完成，否则超时。
readOnly | 默认false。该事务是否只读。
rollbackFor | 用于指定能够触发回滚的异常类型。多个类型以,（英文逗号）隔开
rollbackForClassName | 定义异常的名字，这些异常会触发回滚机制。多个类型以,（英文逗号）隔开
noRollbackFor | 抛出异常，不回滚。多个类型以,（英文逗号）隔开
noRollbackForClassName | 定义异常的名字，抛出异常，不回滚。多个类型以,（英文逗号）隔开

示例：  

```
@Transactional(value = "transactionManager", timeout = 5, rollbackFor = {RuntimeException.class, NullPointerException.class},
    readOnly = true, propagation = Propagation.NOT_SUPPORTED)
public void b() {
    do something...
}
```

### 3. 实现机制
1. spring默认使用AOP扫描被@Transactional的public方法，根据配置信息判断是否由TransactionInterceptor 进行拦截。
2. TransactionInterceptor 进行拦截，在目标方法执行前创建事务，并执行目标方法。
3. 根据sql执行情况，利用抽象事务管理器AbstractPlatformTransactionManager 操作数据源DataSource ，执行提交或者回滚操作。
![image](https://raw.githubusercontent.com/wsk1103/images/master/spring%20transaclational/1.jpg)

### 4. 问题重现
该问题是spring的AOP自调用引起的，注意文字开头说明。

```
public class A {
    a() {
        b();
    }
    //声明事务
    @Transactional
    b() {
        sql操作
    }
}
```

如果这个时候直接通过调用a()方法，那么在b()方法运行错误的时候，是不会回滚代码的。原因如下：  
类A会经过spring 中的AOP生成代理类ProxyA

```
public class Proxy$A {
     A a = new A();
     
     a() {
         a.a();
     }
     
     b() {
        //开启事务
        startTransaction();
         a.b();
     }
     
}
```

然后在运行的时候，是直接调用代理类A（Proxy$A）中的a()方法，该a()方法直接调用原A类的a()方法，所以不会启动事务，最终导致事务失效。

### 5. 解决方法
- 将b()方法抽出来，重新声明一个类，并且该类交由spring管理控制。 
- 同时在a()上添加@Transactional注解或者在类上添加。
- 在原A类中的a()方法，改为 **((A)AopContext.currentProxy).b()**



