---

title: "Elastic Job学习笔记-搭建分布式定时任务"

tags:
  - JAVA
  - 定时任务
  - 学习笔记
---

### 官方文档：  
[http://elasticjob.io/docs/elastic-job-lite/00-overview/](http://elasticjob.io/docs/elastic-job-lite/00-overview/)  
elastic-job学习（网易乐得技术团队，具体说明）  
[http://tech.lede.com/2017/06/23/rd/server/elasticJob/](http://tech.lede.com/2017/06/23/rd/server/elasticJob/)
### 简介    
Elastic-Job-Lite定位为轻量级无中心化解决方案，使用jar包的形式提供最轻量级的分布式任务的协调服务，外部依赖仅Zookeeper。  
使用的时候，只需要在项目中引入相应的maven配置即可。
### 核心理念
#### 2.1 分布式调度
Elastic-Job-Lite并无作业调度中心节点，而是基于部署作业框架的程序在到达相应时间点时各自触发调度。  
注册中心仅用于作业注册和监控信息存储。而主作业节点仅用于处理分片和清理等功能。
#### 2.2 作业高可用
Elastic-Job-Lite提供最安全的方式执行作业。将分片总数设置为1，并使用多于1台的服务器执行作业，作业将会以1主n从的方式执行。  
一旦执行作业的服务器崩溃，等待执行的服务器将会在下次作业启动时替补执行。开启失效转移功能效果更好，可以保证在本次作业执行时崩溃，备机立即启动替补执行。
#### 2.3 最大限度利用资源
Elastic-Job-Lite也提供最灵活的方式，最大限度的提高执行作业的吞吐量。将分片项设置为大于服务器的数量，最好是大于服务器倍数的数量，作业将会合理的利用分布式资源，动态的分配分片项。  
例如：3台服务器，分成10片，则分片项分配结果为服务器A=0,1,2;服务器B=3,4,5;服务器C=6,7,8,9。 如果服务器C崩溃，则分片项分配结果为服务器A=0,1,2,3,4;服务器B=5,6,7,8,9。在不丢失分片项的情况下，最大限度的利用现有资源提高吞吐量。
### 整体架构图  
![image](https://raw.githubusercontent.com/wsk1103/images/master/elastic-job/1.png)

### 主要功能  
#### 分布式
重写Quartz基于数据库的分布式功能，改用Zookeeper实现注册中心。
#### 并行调度
采用任务分片方式实现。将一个任务拆分为n个独立的任务项，由分布式的服务器并行执行各自分配到的分片项。
#### 弹性扩容缩容
将任务拆分为n个任务项后，各个服务器分别执行各自分配到的任务项。一旦有新的服务器加入集群，或现有服务器下线，elastic-job将在保留本次任务执行不变的情况下，下次任务开始前触发任务重分片。
#### 集中管理
采用基于Zookeeper的注册中心，集中管理和协调分布式作业的状态，分配和监听。外部系统可直接根据Zookeeper的数据管理和- 监控elastic-job。
#### 定制化流程型任务
作业可分为简单和数据流处理两种模式，数据流又分为高吞吐处理模式和顺序性处理模式，其中高吞吐处理模式可以开启足够多的线程快速的处理数据，而顺序性处理模式将每个分片项分配到一个独立线程，用于保证同一分片的顺序性。
#### 失效转移
弹性扩容缩容在下次作业运行前重分片，但本次作业执行的过程中，下线的服务器所分配的作业将不会重新被分配。失效转移功能可以在本次作业运行中用空闲服务器抓取孤儿作业分片执行。同样失效转移功能也会牺牲部分性能。
#### 运行时状态收集
监控作业运行时状态，统计最近一段时间处理的数据成功和失败数量，记录作业上次运行开始时间，结束时间和下次运行时间。
#### 作业停止，恢复和禁用
用于操作作业启停，并可以禁止某作业运行（上线时常用）。
#### Spring命名空间支持
elastic-job可以不依赖于spring直接运行，但是也提供了自定义的命名空间方便与spring集成。
#### 运维平台
提供web控制台用于管理作业和注册中心。
#### 稳定性
在服务器无波动的情况下，并不会重新分片；即使服务器有波动，下次分片的结果也会根据服务器IP和作业名称哈希值算出稳定的分片顺序，尽量不做大的变动。
#### 高性能
同一服务器的批量数据处理采用自动切割并多线程并行处理。
#### 灵活性
所有在功能和性能之间的权衡，都可通过配置开启/关闭。如：elastic-job会将作业运行状态的必要信息更新到注册中心。如果作业执行频度很高，会造成大量Zookeeper写操作，而分布式Zookeeper同步数据可能引起网络风暴。因此为了考虑性能问题，可以牺牲一些功能，而换取性能的提升。
幂等性
elastic-job可牺牲部分性能用以保证同一分片项不会同时在两个服务器上运行。
#### 容错性
作业服务器与Zookeeper服务器通信失败则立即停止作业运行，防止作业注册中心将失效的分片分项配给其他作业服务器，而当前作业服务器仍在执行任务，导致重复执行。

### 作业启动流程图  
![image](https://raw.githubusercontent.com/wsk1103/images/master/elastic-job/3.jpeg)

- (1)第一台服务器上线触发主服务器选举。主服务器一旦下线，则重新触发选举，选举过程中阻塞，只有主服务器选举完成，才会执行其他任务。  
- (2)某作业服务器上线时会自动将服务器信息注册到注册中心，下线时会自动更新服务器状态。
- (3)主节点选举，服务器上下线，分片总数变更均更新重新分片标记。
- (4)定时任务触发时，如需重新分片，则通过主服务器分片，分片过程中阻塞，分片结束后才可执行任务。如分片过程中主服务器下线，则先选举主服务器，再分片。
- (5)通过(4)可知，为了维持作业运行时的稳定性，运行过程中只会标记分片状态，不会重新分片。分片仅可能发生在下次任务触发前。
- (6)每次分片都会按服务器IP排序，保证分片结果不会产生较大波动。
- (7)实现失效转移功能，在某台服务器执行完毕后主动抓取未分配的分片，并且在某台服务器下线后主动寻找可用的服务器执行任务。

### 作业执行流程图
![image](https://raw.githubusercontent.com/wsk1103/images/master/elastic-job/4.png)



### 环境说明
Java 1.7 +  
zookeeper 3.4.6 +  
maven 3.0.4 +  
### 引入MAVEN

```
<elastic-job.version>2.1.5</elastic-job.version>
<gson.version>2.6.1</gson.version>
<curator.version>2.11.0</curator.version>

<!--定时任务框架依赖-->
<dependency>
    <artifactId>elastic-job-lite-spring</artifactId>
    <groupId>com.dangdang</groupId>
    <version>${elastic-job.version}</version>
</dependency>
<dependency>
    <artifactId>curator-framework</artifactId>
    <groupId>org.apache.curator</groupId>
    <version>2.11.0</version>
</dependency>
<dependency>
    <artifactId>curator-recipes</artifactId>
    <groupId>org.apache.curator</groupId>
    <version>2.11.0</version>
    <exclusions>
        <exclusion>
            <artifactId>curator-framework</artifactId>
            <groupId>org.apache.curator</groupId>
        </exclusion>
    </exclusions>
</dependency>
<!--定时任务框架依赖结束-->
<!-- 修改dubbo依赖zookeeper版本-->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.6</version>
</dependency>
<dependency>
    <groupId>com.101tec</groupId>
    <artifactId>zkclient</artifactId>
    <version>0.5</version>
</dependency>
```


### 开发说明
新建一个类，实现 SimpleJob或者Dataflow
#### SimpleJob
意为简单实现，未经任何封装的类型。需实现SimpleJob接口。该接口仅提供单一方法用于覆盖，此方法将定时执行。与Quartz原生接口相似，但提供了弹性扩缩容和分片等功能。
#### Dataflow
Dataflow类型用于处理数据流，需实现DataflowJob接口。该接口提供2个方法可供覆盖，分别用于抓取(fetchData)和处理(processData)数据。  
可通过DataflowJobConfiguration配置是否流式处理。  
流式处理数据只有fetchData方法的返回值为null或集合长度为空时，作业才停止抓取，否则作业将一直运行下去； 非流式处理数据则只会在每次作业执行过程中执行一次fetchData方法和processData方法，随即完成本次作业。  
如果采用流式作业处理方式，建议processData处理数据后更新其状态，避免fetchData再次抓取到，从而使得作业永不停止。 流式数据处理参照TbSchedule设计，适用于不间歇的数据处理。  
在com.gw.timer.job中创建相应的类。  
eg。  

```
package com.gw.timer.job;
import com.alibaba.fastjson.JSON;
import com.dangdang.ddframe.job.api.ShardingContext;
import com.dangdang.ddframe.job.api.simple.SimpleJob;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.util.Date;
/**
* @DESCRIPTION : 这是一个测试类
* @AUTHOR : WuShukai
* @TIME : 2018/9/18 15:42
*/
public class TestJob implements SimpleJob {
    private static final Logger log = LoggerFactory.getLogger(TestJob.class);
    @Override
    public void execute(ShardingContext shardingContext) {
        log.info(JSON.toJSONString(shardingContext, true));
        for (int i = 0; i < 5; i++) {
            System.out.println("testJob=====>go: " + i +" , " + new Date());
            try {
                Thread.sleep(1000 * 4);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### 作业启动配置（目前使用spring配置启动）
corn 表达式网站 [http://cron.qqe2.com/](http://cron.qqe2.com/)  
在**spring-job.xml**中声明job，并配置。  

```
<!--测试用例-->
<job:simple registry-center-ref="regCenter" cron="0/30 * * * * ?" sharding-total-count="1" id="testJob"
    class="com.gw.timer.job.TestJob" overwrite="true" failover="true"
job-sharding-strategy-class="com.gw.timer.strategy.FirstJobShardingStrategy"/>
```

### 注册中心配置
#### reg:zookeeper命名空间属性详细说明

|属性名| 类型| 是否必填 |缺省值| 描述|
|---|----|---|---|----|
id |String| 是|  |注册中心在Spring容器中的主键
server-lists| String |是| |  连接Zookeeper服务器的列表包括IP地址和端口号多个地址用逗号分隔如: host1:2181,host2:2181
namespace| String |是 | | Zookeeper的命名空间|
base-sleep-time-milliseconds| int| 否| 1000 |等待重试的间隔时间的初始值单位：毫秒
max-sleep-time-milliseconds |int |否| 3000 |等待重试的间隔时间的最大值单位：毫秒
max-retries| int |否| 3| 最大重试次数
session-timeout-milliseconds |int| 否| 60000 |会话超时时间单位：毫秒
connection-timeout-milliseconds |int| 否 |15000 |连接超时时间单位：毫秒
digest| String |否 | | 连接Zookeeper的权限令牌缺省为不需要权限验证

#### job:simple命名空间属性详细说明

|属性名 |类型| 是否必填 |缺省值| 描述|
|---|----|---|---|----|
id| String| 是 | | 作业名称|
class |String |否 | | 作业实现类，需实现ElasticJob接口
job-ref| String| 否| |  作业关联的beanId，该配置优先级大于class属性配置
registry-center-ref |String |是| |  注册中心Bean的引用，需引用reg:zookeeper的声明
cron |String |是 | |cron表达式，用于控制作业触发时间
sharding-total-count |int |是  | |作业分片总数
sharding-item-parameters| String |否 | | 分片序列号和参数用等号分隔，多个键值对用逗号分隔分片序列号从0开始，不可大于或等于作业分片总数如：0=a,1=b,2=c
job-instance-id |String| 否 |defaultInstance| 作业实例主键，同IP可运行实例主键不同, 但名称相同的多个作业实例
job-parameter| String| 否| |  作业自定义参数作业自定义参数，可通过传递该参数为作业调度的业务方法传参，用于实现带参数的作业例：每次获取的数据量、作业实例从数据库读取的主键等
monitor-execution |boolean |否 |true| 监控作业运行时状态每次作业执行时间和间隔时间均非常短的情况，建议不监控作业运行时状态以提升效率。因为是瞬时状态，所以无必要监控。请用户自行增加数据堆积监控。并且不能保证数据重复选取，应在作业中实现幂等性。每次作业执行时间和间隔时间均较长的情况，建议监控作业运行时状态，可保证数据不会重复选取。
monitor-port |int| 否| -1 |作业监控端口建议配置作业监控端口, 方便开发者dump作业信息。使用方法: echo “dump” \| nc 127.0.0.1 9888
max-time-diff-seconds |int| 否 |-1| 最大允许的本机与注册中心的时间误差秒数如果时间误差超过配置秒数则作业启动时将抛异常配置为-1表示不校验时间误差
failover| boolean |否 |false| 是否开启失效转移
misfire| boolean |否 |true |是否开启错过任务重新执行
job-sharding-strategy-class |String |否 | | 作业分片策略实现类全路径默认使用平均分配策略详情参见：作业分片策略
description |String |否 | | 作业描述信息
disabled| boolean |否 |false| 作业是否禁止启动可用于部署作业时，先禁止启动，部署结束后统一启动
overwrite |boolean| 否| false |本地配置是否可覆盖注册中心配置如果可覆盖，每次启动作业都以本地配置为准
job-exception-handler |String |否 | | 扩展异常处理类
executor-service-handler| String |否 | |扩展作业处理线程池类
reconcile-interval-minutes |int| 否| 10| 修复作业服务器不一致状态服务调度间隔时间，配置为小于1的任意值表示不执行修复单位：分钟
event-trace-rdb-data-source |String| 否 | | 作业事件追踪的数据源Bean引用


#### job:dataflow命名空间属性详细说明

|属性名 |类型| 是否必填 |缺省值| 描述|
|---|----|----|----|----|
streaming-process| boolean| 否 |false| 是否流式处理数据如果流式处理数据, 则fetchData不返回空结果将持续执行作业如果非流式处理数据, 则处理数据完成后作业结束

### 部署指南
#### 应用部署
- a. 启动Elastic-Job-Lite指定注册中心的Zookeeper。
- b. 运行包含Elastic-Job-Lite和业务代码的jar文件。不限与jar或war的启动方式。
#### 运维平台部署(可选)
- a.  编译gw-web-job-console.tar.gz，可通过mvn install编译获取。
- b.  解压缩gw-web-job-console.tar.gz并执行bin\start.sh。
- c.  打开浏览器访问http://localhost:8899/即可访问控制台。8899为默认端口号，可通过启动脚本输入-p自定义端口号。

#### 运维平台功能
- a.  登录安全控制
- b.  注册中心、事件追踪数据源管理
- c.  快捷修改作业设置
- d.  作业和服务器维度状态查看
- e.  操作作业禁用\启用、停止和删除等生命周期
- f.  事件追踪查询



### 13.遇到的问题
主要遇到的问题是MAVEN版本冲突，看第8看的修改。
- 1.系统使用的zookeeper为3.4.5，而elastic-job建议使用的zookeeper版本为3.4.6。  
所以系统dubbo在引入zookeeper的时候，pom使用的版本为3.4.5，因此需要修改为3.4.6，并且相应的 zkclient 版本修改为0.5
- 2.系统使用的cat中引入的gson为1.6，所以需要在项目中强制修改gson版本为2.6.1，否则会出现依赖冲突。
- 3.elastic-job中， curator默认使用的版本为2.10.0，此时会出现的WARN ：The version of ZooKeeper being used doesn't support Container nodes. CreateMode.PERSISTENT will be used instead. 所以需要将curator的版本修改为2.11.0。
    

