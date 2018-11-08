---
title: "Spring Data + JSF学习笔记"

tags:
  - Java
  - 学习笔记
---


## 开发工具 IDEA 2017
## 使用框架JSF+Spring Data
## 插件 Lombok

JSF标签控件效果预览可以查看  [https://www.primefaces.org/showcase/](https://www.primefaces.org/showcase/)

## Spring Data作用
操作数据库，该框架直接封装了许多方法，可以直接使用使用操作数据库，不需要自己动手编写SQL，但是对于比较复杂的SQL查询，可以自定义SQL查询。

## 使用方法
#### 1. 根据数据库相应的表创建相应的实体类（Entity）
例如

```
package com.wsk.business.entity.demo;

import lombok.Data;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;

@Data
@Entity
@Table(name = "TB_LOAN_DEMO")
public class Demo{
    @Id
    private int id;
    private String name;
    private String password;
}
```


#### 2. 定义DAO层

```
package com.wsk.business.repository.demo;


import com.wsk.business.entity.demo.Demo;
import com.wsk.business.repository.demo.custom.DemoRepositoryCustom;
import com.wsk.core.repository.JPAObjectRepository;
import com.wsk.vo.request.demo.DemoSearchRequest;
import com.wsk.vo.response.demo.DemoSearchResponse;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

/*
 * 1.Repository是一个空接口，即是一个标记接口
 * 2.若我们定义的接口继承了Repository，则该接口会被IOC容器识别为一个Repository Bean
 * 注入到IOC容器中，进而可以在该接口中定义满足一定规则的接口
 * 3.实际上也可以通过一个注解@RepositoryDefination 注解来替代Repository接口
 */
/*
 * 在Repository子接口中声明方法
 * 1.不是随便声明的，而需要符合一定的规范
 * 2.查询方法以find|read|get开发
 * 3.涉及条件查询，条件的属性需要定义关键字连接
 * 4.要注意的额是，条件的属性以字母大写
 * 5.支持属性的级联查询，若当前类有符合条件的属性，则优先使用，则不使用级联属性
 * 若需要使用级联属性，则属性之间使用——进行连接
 * */

 /**
*JPAObjectRepository是已经封装好的接口，该接口已经间接继承Repository接口，
*/

public interface DemoRepository extends JPAObjectRepository<Demo, Integer> {
    //通过id查询
    Demo findById(Integer id);

    //根据name来查询
    //where name = ?
    Demo findByName(String name);

    //like查询和id
    //where name like ?% and id < ?
    List<Demo> findByNameStartingWithAndIdLessThan(String name, Integer id);

    //like查询和id
    //where name like %? and id > ?
    List<Demo> findByNameEndingWithAndIdGreaterThan(String name, Integer id);

    //in查询 和 OR查询
    //where name in（？, ? , ?) or id < ?
    List<Demo> findByNameInOrIdLessThan(List<String> name, Integer id);

    //自定义查询
    //查询id值最大的那个demo
    //使用@Query注解可以自定义JPQL语句，语句可以实现更灵活的查询
    @Query("SELECT  d FROM demo d WHERE d.id=(SELECT max(d2.id) FROM demo d2)")
    Demo getMaxIdPerson();

    //使用占位符
    @Query("select d from demo d where id = :id")
    Demo getOneById(@Param("id") Integer id);

    //设置nativeQuery=true 可以使用原生的sql查询
    @Query(value = "select * from demo" ,nativeQuery = true)
    Demo getOneById();

    //可以通过自定义的JPQL 完成update和delete操作，注意：JPQL不支持Insert操作
    //在@Query注解中编写JPQL语句，但必须使用@Modify进行修饰，以通知SpringData，这是一个Update或者Delete
    //Update或者delete操作，需要使用事务，此时需要定义Service层，在service层的方法上添加事务操作
    //默认情况下，SpringData的每个方法上有事务，但都是一个只读事务，他们不能完成修改操作
    @Modifying
    @Query("update d from demo d where d.id = :id")
    Demo updateOneById(@Param("id") Integer id);

    //分页查询
    Page<DemoSearchResponse> list(DemoSearchRequest demoSearchRequest, Pageable pageable);


}
```
接下来就可以直接使用该接口操作数据库

#### 3. 编写Serivce层

```
package com.wsk.business.service.demo;

import com.wsk.business.entity.demo.Demo;
import com.wsk.core.service.BaseObjectService;

import java.util.List;

public interface DemoService extends BaseObjectService<Demo, Integer> {

    List<Demo> findAll();

    Demo findOneById(Integer id);

    .....
}


```

#### 4. 实现Service接口

```
package com.wsk.business.service.demo.impl;

import com.wsk.business.entity.demo.Demo;
import com.wsk.business.repository.demo.DemoRepository;
import com.wsk.business.service.demo.DemoService;
import com.wsk.core.repository.JPAObjectRepository;
import com.wsk.core.service.BaseObjectServiceImpl;
import com.wsk.core.vo.PageSearcher;
import com.wsk.vo.request.demo.DemoSearchRequest;
import com.wsk.vo.response.demo.DemoSearchResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.retry.RetryException;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

/*
该接口一般都需要继承BaseObjectServiceImpl接口，
BaseObjectServiceImpl接口封装好了许多常用的功能
 */
@Service
@Transactional
@Retryable(value = { RetryException.class })
public class DemoServiceImpl extends BaseObjectServiceImpl<Demo, Integer> implements DemoService {

    @Autowired
    private DemoRepository demoRepository;

    //返回实体对象
    @Override
    protected JPAObjectRepository<Demo, Integer> getRepository() {
        return demoRepository;
    }

    public Page<DemoSearchResponse> list(PageSearcher<DemoSearchRequest> pageSearcher) {
        return demoRepository.list(pageSearcher.getSearchCondition(), pageSearcher.getPageable());
    }
    //查询所有demo对象
    @Override
    public List<Demo> findAll(){
        return demoRepository.findAll();
    }

    @Override
    public Demo findOneById(Integer id) {
        return demoRepository.findById(id);
    }

}


```

#### 5. 定义业务处理层。
在web项目中定义bean，用于与JSF页面交互。

```
package com.wsk.web.bean.demo;

import com.wsk.business.entity.demo.Demo;
import com.wsk.business.service.demo.DemoService;
import com.wsk.core.aspect.Debug;
import com.wsk.vo.request.demo.DemoSearchRequest;
import com.wsk.vo.response.demo.DemoSearchResponse;
import com.wsk.web.configuration.primefaces.JSFListingBean;
import lombok.Data;
import lombok.EqualsAndHashCode;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import javax.faces.bean.ManagedBean;
import javax.faces.bean.ViewScoped;
import java.util.List;

@Data
@Debug
@Component
@ViewScoped
@EqualsAndHashCode(callSuper = true)
@ManagedBean(name = "demoBean")//定义JSF页面中需要被引用的bean的名称
public class DemoBean extends JSFListingBean<DemoSearchRequest, DemoSearchResponse> {

    @Autowired
    private DemoService demoService;
    private List<Demo> demos;

    //返回数据，返回的数据对应JSF页面中的value
    @PostConstruct
    public void init(){
        demos = demoService.findAll();
    }

}

```
至此，后台服务基本完成

## 接下来是JSF页面
页面存放在web项目的views文件下


JSF标签控件效果预览可以查看  [https://www.primefaces.org/showcase/](https://www.primefaces.org/showcase/)

```
<ui:composition xmlns="http://www.w3.org/1999/xhtml"
                xmlns:ui="http://java.sun.com/jsf/facelets"
                xmlns:h="http://java.sun.com/jsf/html"
                xmlns:p="http://primefaces.org/ui"
                xmlns:f="http://java.sun.com/jsf/core"
                xmlns:constants="http://www.xmiles.cn/constants/tags"
                xmlns:security="http://www.springframework.org/security/tags"
                template="/templates/template.xhtml">
    <ui:define name="title">DEMO</ui:define>
    <ui:define name="content">
        <h:form>
        <div class="card">
            <p:dataTable id="userDataTable"
                         style="width: 100%"
                         var="demo"
                         value="#{demoBean.demos}"
                         rows="#{constants:get('ROWS')}"
                         lazy="true"
                         paginator="true"
                         paginatorTemplate="#{constants:get('PAGINATOR_TEMPLATE')}"
                         rowsPerPageTemplate="#{constants:get('ROWS_PER_PAGE_TEMPLATE')}"
                         emptyMessage="#{constants:get('EMPTY_MESSAGE')}"
                         scrollable="true"
                         scrollWidth="97%"
                         reflow="true"
                         selection="#{demoBean.selectedRecords}"
                         resizableColumns="true"
                         >
                <p:column selectionMode="multiple" style="width:16px;text-align:center"
                          rendered="#{demoBean.isBatch()}"/>

                <p:column headerText="id">
                    <h:outputText value="#{demo.id}"/>
                </p:column>
                <p:column headerText="姓名">
                    <h:outputText value="#{demo.name}"/>
                </p:column>
                <p:column headerText="密码">
                    <h:outputText value="#{demo.password}"/>
                </p:column>
            </p:dataTable>
        </div>
        </h:form>
    </ui:define>
    <!--</h:form>-->
</ui:composition>
```
