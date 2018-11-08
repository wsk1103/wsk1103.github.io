---

title: "自定义Java定时器（基于ScheduledExecutorService)"

tags:
  - JAVA
  - 定时任务

---


## JAVA版本：jdk1.8，代码中有使用Lambda语法糖。
## 数据库：MySQL
## 框架：Spring Data
## 开发工具：IDEA 2017.3.2
## Lombok
## PS：
#### 1. 主要是结合Spring Boot一起使用，并在Spring Boot启动的时候一起启动运行。
#### 2. 和数据库结合使用的主要目的是在程序运行的时候，可以通过操作数据库对定时任务的控制，例如关闭和启动任务，添加任务，修改定时时间等等。
#### 3. 查看所有任务，错误日志和运行日志等等。
#### 4. 该定时器最初被设计的目的是用于每天晚上0点定时爬取某音乐平台的热门音乐绑定，新音乐榜单等信息。

### 使用方法：
##### 1. 创建相应的数据库表（后续会自动创建）
##### 2. 创建一个Runnable继承MyRunnable抽象类
##### 3. 将该类的全名（包括包名）存储到表中，并插入相应的任务名称，开始时间和时间表达式，即可。

---
---------------------------------------
---

## 1. 总体思路
	1.1 使用一个线程查询数据库，将还未完成的任务查找出来（每5s查一次）
	1.2 将结果存储到阻塞队列中（通过反射获取runnable）
	1.3 另外的线程循环获取阻塞队列中的runnable，
	1.4 在运行runnable之前，更新数据库
	1.5 运行runnable
	1.6 写入日志
	1.7 如果出错，写入错误日志
## 2. 数据库结构

任务列表库  
![image](https://raw.githubusercontent.com/wsk1103/images/master/mytimer/1.png)
	
```
	DROP TABLE IF EXISTS `mytask`;
	CREATE TABLE `mytask` (
	  `id` int(11) NOT NULL AUTO_INCREMENT,
	  `taskname` varchar(255) NOT NULL COMMENT '任务名称',
	  `status` int(1) DEFAULT NULL COMMENT '状态',
	  `starttime` datetime DEFAULT NULL COMMENT '开始时间',
	  `nexttime` datetime DEFAULT NULL COMMENT '下一次运行时间',
	  `begintime` datetime DEFAULT NULL COMMENT '起始时间',
	  `classname` varchar(255) DEFAULT NULL COMMENT '类名，必须是全名，包含包名',
	  `expression` varchar(20) DEFAULT NULL COMMENT '时间表达式',
	  PRIMARY KEY (`id`),
	  UNIQUE KEY `index_id` (`id`) USING BTREE,
	  UNIQUE KEY `index_taskname` (`taskname`) USING BTREE
	) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
	# 插入数据
	INSERT INTO `mytask` VALUES ('1', 'hotting_music', '1', '2018-01-28 13:26:51', '2018-01-29 13:26:51', '2018-01-27 22:49:35', 'com.wsk.movie.task.runnable.MusicHottingTaskRunnable', '00 00 00 1');
	INSERT INTO `mytask` VALUES ('2', 'hot_music', '1', '2018-01-28 00:52:42', '2018-01-28 00:52:54', '2018-01-28 13:53:03', 'com.wsk.movie.task.runnable.MusicHotTaskRunnable', '00 00 00 1');
	INSERT INTO `mytask` VALUES ('3', 'new_music', '1', '2018-01-28 00:53:53', '2018-01-28 00:54:02', '2018-01-28 13:54:00', 'MusicNewTaskRunnable', '00 00 00 1');

```
	
	
错误信息库  
![这里写图片描述](https://raw.githubusercontent.com/wsk1103/images/master/mytimer/2.png)  

```
	DROP TABLE IF EXISTS `mytaskerror`;
	CREATE TABLE `mytaskerror` (
	  `id` int(11) NOT NULL AUTO_INCREMENT,
	  `taskname` varchar(255) DEFAULT NULL COMMENT '定时任务名',
	  `rtime` datetime DEFAULT NULL COMMENT '发生时间',
	  `msg` varchar(2000) DEFAULT NULL COMMENT '错误信息',
	  `classname` varchar(255) DEFAULT NULL COMMENT '类名',
	  PRIMARY KEY (`id`)
	) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

	

	
日志列表库  
![这里写图片描述](https://raw.githubusercontent.com/wsk1103/images/master/mytimer/3.png)
```
	DROP TABLE IF EXISTS `mytasklog`;
	CREATE TABLE `mytasklog` (
	  `id` int(11) NOT NULL AUTO_INCREMENT,
	  `taskname` varchar(255) DEFAULT NULL,
	  `rtime` datetime DEFAULT NULL,
	  `classname` varchar(255) DEFAULT NULL,
	  PRIMARY KEY (`id`)
	) ENGINE=InnoDB AUTO_INCREMENT=19 DEFAULT CHARSET=utf8;
	
```
	
## 3. 项目结构
	task
		entity:数据库实体类
		queue:存储runnable的队列
		runnable:运行的类
		service:操作数据库
		tool:工具类-主要是日期表达式的转化

		
![这里写图片描述](https://raw.githubusercontent.com/wsk1103/images/master/mytimer/4.png)
## 4. 实操

#### 1. 执行sql语句，创建task相关的表
#### 2. 使用idea连接数据库，并根据数据库task相关的表生成对应的实体，存放于entity包中

![这里写图片描述](https://raw.githubusercontent.com/wsk1103/images/master/mytimer/5.png)
#### 3. 创建service包和相应的接口
###### 3.1 错误日志记录接口

```
package com.wsk.movie.task.service;

import com.wsk.movie.task.entity.MytaskerrorEntity;
import org.springframework.data.jpa.repository.JpaRepository;

/**
 * @DESCRIPTION :错误任务记录
 * @AUTHOR : WSK1103
 * @TIME : 2018/1/24  22:43
 */
public interface MyErrorTaskRepository extends JpaRepository<MytaskerrorEntity, Integer> {
}

```

###### 3.2 平时日志记录接口

```
package com.wsk.movie.task.service;

import com.wsk.movie.task.entity.MytasklogEntity;
import org.springframework.data.jpa.repository.JpaRepository;

/**
 * @DESCRIPTION :平时日志记录
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/27  22:39
 */
public interface MyTaskLogRepository extends JpaRepository<MytasklogEntity, Integer> {
}


```
###### 3.3 任务表接口

```
package com.wsk.movie.task.service;

import com.wsk.movie.task.entity.MytaskEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import javax.transaction.Transactional;
import java.sql.Timestamp;
import java.util.Date;
import java.util.List;

/**
 * @DESCRIPTION :任务表接口
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/23  23:02
 */
public interface MyTaskRepository extends JpaRepository<MytaskEntity, Integer> {
    MytaskEntity findByTaskname(String taskname);

    //更新
    @Transactional
    @Query(value = "update MytaskEntity m set m.starttime = :start,m.nexttime = :next where m.taskname = :name")
    @Modifying
    void updateTime(@Param("name") String name, @Param("start") Date start, @Param("next") Date next);

    //更新
    @Transactional
    @Query(value = "update MytaskEntity m set m.starttime = :start,m.nexttime = :next where m.taskname = :name")
    @Modifying
    void updateTime(@Param("name") String name, @Param("start") Timestamp start, @Param("next") Timestamp next);

    //根据任务名关闭
    @Transactional
    @Query(value = "update MytaskEntity m set m.status = :status where m.taskname = :name")
    @Modifying
    void updateStatus(@Param("name") String name, @Param("status") int status);

    //根据id关闭
    @Transactional
    @Query(value = "update MytaskEntity m set m.status = :status where m.taskname = :id")
    @Modifying
    void updateStatus(@Param("id") int id, @Param("status") int status);

	//待执行的任务
    @Query(value = "select m from MytaskEntity m where m.status = 1")
    List<MytaskEntity> starts();

}

```
#### 4.创建MyRunnable抽象类，该类继承Runnable，自定义的线程都必须继承该类

```
package com.wsk.movie.task.runnable;

import lombok.Data;

/**
 * @DESCRIPTION :
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/23  23:05
 */
@Data
public abstract class MyRunnable implements Runnable {
}

```
###### 4.1  创建几个运行测试

```
package com.wsk.movie.task.runnable;

import com.wsk.movie.music.HttpUnits;

import java.io.IOException;

/**
 * @DESCRIPTION :音乐定时器,云音乐热歌榜
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/27  13:48
 */
public class MusicHotTaskRunnable extends MyRunnable {

    @Override
    public void run() {
        try {
			//HttpUnits是自定义的一个读取json的工具类
			System.out.println(HttpUnits.urlToString("http://localhost:8080/search/music/hot/1").toString());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```

###### 4.2 创建删除队列中已经运行过的队列key，使数据库查询的相应任务可以添加到队列中

```
package com.wsk.movie.task.runnable;

import com.wsk.movie.task.entity.MytaskEntity;
import com.wsk.movie.task.queue.MyQueue;

/**
 * @DESCRIPTION :
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/29  21:16
 */
public class DelKeyRunnable extends MyRunnable {

    private MytaskEntity entity;

    public DelKeyRunnable(MytaskEntity entity) {
        this.entity = entity;
    }

    @Override
    public void run() {
        System.out.println("del:" + entity.getTaskname());
        MyQueue.getInstance().removeKey(entity.getTaskname());
    }
}

```


#### 5. 创建queue包和对应的队列
###### 5.1 创建MyQueue类，该类是用来存储任务的，而且是单例模式

```
package com.wsk.movie.task.queue;

import lombok.EqualsAndHashCode;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.LinkedTransferQueue;

/**
 * @DESCRIPTION :队列，用来存储任务,单例
 * ArrayBlockingQueue ：一个由数组结构组成的有界阻塞队列。
 * LinkedBlockingQueue ：一个由链表结构组成的有界阻塞队列。
 * PriorityBlockingQueue ：一个支持优先级排序的无界阻塞队列。
 * DelayQueue：一个使用优先级队列实现的无界阻塞队列。
 * SynchronousQueue：一个不存储元素的阻塞队列。
 * LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
 * LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/24  20:53
 */
@EqualsAndHashCode
public class MyQueue {
    //使用无界限阻塞队列
    private LinkedTransferQueue<MyQueueBean> queue;
    //存放数据库唯一名称,根据名称判断任务是否重复
    private static final List<String> LIST = new ArrayList<>();

    private MyQueue() {
        queue = new LinkedTransferQueue<>();
    }

    private static class NestClass {
        private static final MyQueue QUEUE = new MyQueue();
    }

    public static MyQueue getInstance() {
        return NestClass.QUEUE;
    }

    public void offer(MyQueueBean bean) {
	    //加锁，在多线程的情况下防止多加任务
        synchronized (LIST) {
            if (LIST.contains(bean.getEntity().getTaskname())) {
                System.out.println("重复" + bean.getRunnable().getClass().getName());
                return;
            }
        }
        LIST.add(bean.getEntity().getTaskname());
        queue.offer(bean);
//        LIST.forEach(System.out::println);
    }

    public MyQueueBean take() throws InterruptedException {
        //阻塞获取
        return queue.take();
    }

    public void removeKey(MyQueueBean bean){
        LIST.remove(bean.getEntity().getTaskname());
    }

    public void removeKey(MytaskEntity entity) {
        LIST.remove(entity.getTaskname());
    }

    public void removeKey(int id){
        LIST.remove(id);
    }

    public boolean hasNext() {
        return queue.iterator().hasNext();
    }

    public int size(){
        return queue.size();
    }

}


```
###### 5.2 创建MyQueueBean，存储任务和任务的属性，队列存储的是该类，线程运行的是该类中的runnable

```
package com.wsk.movie.task.queue;

import com.wsk.movie.task.entity.MytaskEntity;
import com.wsk.movie.task.runnable.MyRunnable;
import lombok.Data;
import lombok.experimental.Accessors;

/**
 * @DESCRIPTION :存储任务和任务的属性
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/27  23:57
 */
@Accessors(chain = true)
@Data
public class MyQueueBean {
    private MyRunnable runnable;
    private MytaskEntity entity;
}

```
#### 6. 创建时间表达式解析类

```
package com.wsk.movie.task.tool;

import com.wsk.movie.tool.Tool;

import java.sql.Timestamp;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

/**
 * @DESCRIPTION :将时间表达式转化为秒，自定义的定时任务中，都是以秒为单位运行的
 * 时间表达式
 * 1:00 00 00 00-中间以空格分开
 *  :秒 分 时 日
 * 2:yyyy-MM-dd HH:mm:ss
 * 3:yyyy-MM-dd
 * 4.时间戳Timestamp
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/24  23:21
 */
public class TimeTransform {

    public static SimpleDateFormat day = new SimpleDateFormat("yyyy-MM-dd");
    public static SimpleDateFormat fullDay = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    public static long getTime(String expression) {
        if (Tool.getInstance().isNullOrEmpty(expression)) {
            throw new RuntimeException("不是正确的时间表达式");
        }
        String[] times = expression.split(" ");
        if (times.length > 4) {
            throw new RuntimeException("不是正确的时间表达式");
        }
        //最后总时间
        long start = 0;
        try {
            for (int i = 0; i < times.length; i++) {
                long time = Integer.parseInt(times[i]);
                switch (i) {
                    //秒
                    case 0:
                        start += time;
                        break;
                    //分
                    case 1:
                        start += 60 * time;
                        break;
                    //时
                    case 2:
                        start += 60 * 60 * time;
                        break;
                    //日
                    case 3:
                        start += 60 * 60 * 24 * time;
                }
            }
        } catch (Exception e) {
            try {
                start = fullDay.parse(expression).getTime() - new Date().getTime();
                start = start > 0 ? start / 1000 : 0;
            } catch (ParseException e1) {
                try {
                    start = day.parse(expression).getTime() - new Date().getTime();
                    start = start > 0 ? start / 1000 : 0;
                } catch (ParseException e2) {
                    throw new RuntimeException("不是正确的时间表达式");
                }
            }
        }
        return start;
    }

    public static long getTime(Date date) {
        //最后总时间
        long start;
        start = date.getTime() - new Date().getTime();
        start = start > 0 ? start / 1000 : 0;
        return start;
    }

    public static long getTime(long date) {
        long start;
        start = date - new Date().getTime();
        start = start > 0 ? start / 1000 : 0;
        return start;
    }

    public static long getTime(Timestamp date) {
        long start;
        start = date.getTime() - new Date().getTime();
        start = start > 0 ? start / 1000 : 0;
        return start;
    }
}

```

#### 7. 创建MyTask运行类，该类是主要的线程运行核心

```
package com.wsk.movie.task;

import com.wsk.movie.task.entity.MytaskEntity;
import com.wsk.movie.task.entity.MytaskerrorEntity;
import com.wsk.movie.task.entity.MytasklogEntity;
import com.wsk.movie.task.queue.MyQueue;
import com.wsk.movie.task.queue.MyQueueBean;
import com.wsk.movie.task.runnable.DelKeyRunnable;
import com.wsk.movie.task.runnable.MyRunnable;
import com.wsk.movie.task.service.MyErrorTaskRepository;
import com.wsk.movie.task.service.MyTaskLogRepository;
import com.wsk.movie.task.service.MyTaskRepository;
import com.wsk.movie.task.tool.TimeTransform;
import com.wsk.movie.tool.SpringContextUtil;

import java.sql.Timestamp;
import java.util.Date;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * @DESCRIPTION :自定义定时器-单例
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/23  22:22
 */
public class MyTask implements Runnable{
    /**
     * 使用定时线程池
     * 根据CPU进行任务调度
     */
    private static ScheduledExecutorService service = Executors.newScheduledThreadPool(Runtime.getRuntime().availableProcessors());
    private static MyErrorTaskRepository errorTaskRepository = (MyErrorTaskRepository) SpringContextUtil.getBean(MyErrorTaskRepository.class);
    private static MyTaskLogRepository logRepository = (MyTaskLogRepository) SpringContextUtil.getBean(MyTaskLogRepository.class);
    private static MyTaskRepository repository = (MyTaskRepository) SpringContextUtil.getBean(MyTaskRepository.class);

    private MyTask() {
    }

    @Override
    public void run() {
        execute();
    }

    private static class NestClass {
        private static final MyTask MY_TASK = new MyTask();
    }

    public static MyTask getInstance() {
        return NestClass.MY_TASK;
    }

    public void execute(MyQueue queue) {
        MyQueueBean bean;
        Date now = new Date();
        try {
            bean = queue.take();
//            System.out.println("run:" + bean.getEntity().getTaskname());
        } catch (InterruptedException e) {
            e.printStackTrace();
            MytaskerrorEntity entity = new MytaskerrorEntity();
            entity.setTaskname("");
            entity.setMsg("队列获取失败");
            entity.setRtime(new Timestamp(now.getTime()));
            entity.setClassname("");
            errorTaskRepository.save(entity);
            return;
        }
        MyRunnable runnable = bean.getRunnable();
        MytaskEntity entity = bean.getEntity();
        //更新数据库
        long next = TimeTransform.getTime(entity.getExpression());
        repository.updateTime(entity.getTaskname(), new Timestamp(now.getTime()), new Timestamp(now.getTime() + next * 1000));
        //开始定时任务
        service.scheduleAtFixedRate(runnable, TimeTransform.getTime(entity.getStarttime()), next, TimeUnit.SECONDS);
        //删除队列中的key
        service.schedule(new DelKeyRunnable(entity), next, TimeUnit.SECONDS);
        //日志
        MytasklogEntity log = new MytasklogEntity();
        log.setTaskname(entity.getTaskname());
        log.setClassname(entity.getClassname());
        log.setRtime(new Timestamp(now.getTime()));
        logRepository.save(log);
        System.out.println("队列剩余：" + MyQueue.getInstance().size());
    }

    //开始运行定时任务
    public void execute() {
        while (true) {
            System.out.println("开始定时任务,size:" + MyQueue.getInstance().size());
            execute(MyQueue.getInstance());
        }
    }

    //关闭线程
    public void shutdown() {
        service.shutdown();
    }

}



```

#### 8. 创建一个在Spring Boot启动的时候也启动定时器的任务

```
package com.wsk.movie.task;

import com.wsk.movie.task.entity.MytaskEntity;
import com.wsk.movie.task.entity.MytaskerrorEntity;
import com.wsk.movie.task.queue.MyQueue;
import com.wsk.movie.task.queue.MyQueueBean;
import com.wsk.movie.task.runnable.MyRunnable;
import com.wsk.movie.task.service.MyErrorTaskRepository;
import com.wsk.movie.task.service.MyTaskRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

import java.sql.Timestamp;
import java.util.Date;
import java.util.List;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * @DESCRIPTION :随着程序的启动而启动运行,运行定时任务，每5秒查询一次数据库
 * @AUTHOR : WuShukai1103
 * @TIME : 2018/1/24  22:11
 */
@Component
public class MainTask implements CommandLineRunner {

    private static final ScheduledExecutorService SCHEDULED_EXECUTOR_SERVICE = Executors.newScheduledThreadPool(2);

    private final MyTaskRepository repository;
    private final MyErrorTaskRepository errorTaskRepository;

    @Autowired
    public MainTask(MyTaskRepository repository, MyErrorTaskRepository errorTaskRepository) {
        this.repository = repository;
        this.errorTaskRepository = errorTaskRepository;
    }

    @Override
    public void run(String... strings) throws Exception {
        SCHEDULED_EXECUTOR_SERVICE.scheduleAtFixedRate(() -> {
            List<MytaskEntity> entities = repository.starts();
//            System.out.println("查询数据库:" + TimeTransform.fullDay.format(new Date()) + ",size:" + entities.size());
            for (MytaskEntity e : entities) {
                Timestamp now = new Timestamp(new Date().getTime());
                MyRunnable runnable;
                try {
                    //通过反射获取运行的类
                    runnable = (MyRunnable) Class.forName(e.getClassname()).newInstance();
                    //将runnable存储到队列中
                    MyQueue.getInstance().offer(new MyQueueBean().setEntity(e).setRunnable(runnable));
                } catch (ClassNotFoundException | InstantiationException | IllegalAccessException ex) {
                    ex.printStackTrace();
                    MytaskerrorEntity entity = new MytaskerrorEntity();
                    entity.setClassname(e.getClassname());
                    entity.setMsg(ex.getMessage());
                    entity.setRtime(now);
                    entity.setTaskname(e.getTaskname());
                    errorTaskRepository.save(entity);
                }
            }
        }, 0, 5, TimeUnit.SECONDS);
        Thread.sleep(1);
        //加载队列后，立即运行MyTask
        SCHEDULED_EXECUTOR_SERVICE.execute(MyTask.getInstance());
    }
}

```
### 自此，一个自定义的定时任务就完成了。