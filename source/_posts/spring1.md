title: spring基础:spring与ibatis简单demo
date: 2015-02-25 19:16:21
tags: spring
---
##工程目录
建立一个test工程，marven作为依赖管理，具体工程目录如图所示：
<!-- more -->

![][1]

##web.xml配置
```xml
<web-app version="2.4"
	xmlns="http://java.sun.com/xml/ns/j2ee" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee 
	http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">
	<display-name>Spring MVC Application</display-name>

    <servlet>
		<servlet-name>mvc-dispatcher</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>mvc-dispatcher</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>
	<!-- 强制转换成utf8格式，防止中文乱码-->
	<filter>
		<filter-name>characterEncodingFilter</filter-name>
		<filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
		<init-param>
			<param-name>encoding</param-name>
			<param-value>UTF-8</param-value>
		</init-param>
		<init-param>
			<param-name>forceEncoding</param-name>
			<param-value>true</param-value>
		</init-param>
	</filter>
	<filter-mapping>
		<filter-name>characterEncodingFilter</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>
</web-app>
```
![DispatchServlet][2]




##mvc-dispatcher-servlet.xml
```
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="com.springapp.mvc"/>
    
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/pages/"/>
        <property name="suffix" value=".jsp"/>
    </bean>

    <bean id="dataSource"  class="org.apache.commons.dbcp.BasicDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/ibatis"/>
        <property name="username" value="root"/>
        <property name="password" value="root"/>
    </bean>

    <bean id="sqlMapClient" class="org.springframework.orm.ibatis.SqlMapClientFactoryBean">
        <property name="configLocation" value="classpath:SqlMapConfig.xml"/>
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <bean id="Person" class="po.Person">
    </bean>

    <bean id = "Action" class="dao.impl.ActionImpl">
    <property name="sqlMapClient" ref="sqlMapClient"></property>
    </bean>

</beans>
```


##SqlMapConfig.xml
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE sqlMapConfig
        PUBLIC "-//iBATIS.com//DTD SQL Map Config 2.0//EN"
        "http://www.ibatis.com/dtd/sql-map-config-2.dtd">

<sqlMapConfig>
    <settings useStatementNamespaces="true"/>
    <!--数据源配置  这块用 BD2数据库 -->
    <sqlMap resource="ibatis/Person.xml" />
</sqlMapConfig>
```

##Person.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE sqlMap
        PUBLIC "-//iBATIS.com//DTD SQL Map 2.0//EN"
        "http://www.ibatis.com/dtd/sql-map-2.dtd">
<sqlMap  namespace="ibatis.Person">
    <typeAlias alias="person" type="po.Person" />

    <insert id="insert" parameterClass="po.Person">
	       insert into person(name, sex) values (#name#,#sex#)
    </insert>

</sqlMap>
```

##Person.java
```java
package po;
import java.io.Serializable;
public class Person implements Serializable{
    private static final long serialVersionUID = -517413165963030507L;
    
    private String name;
    private int sex;
    public Person(){
    }
    public Person(String name,int sex){
        this.name = name;
        this.sex = sex;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getSex() {
        return sex;
    }
    public void setSex(int sex) {
        this.sex = sex;
    }

}
```

##IAction.java
```
package dao;
import po.Person;
public interface IAction {
    public Long insertPerson(Person person);   //添加
}
```

##ActionImpl.java
```java
package dao.impl;
import org.springframework.orm.ibatis.SqlMapClientTemplate;
import po.Person;
import dao.IAction;

public class ActionImpl extends SqlMapClientTemplate implements IAction {
    public ActionImpl() {
    }
    //添加操作
    @Override
    public Long insertPerson(Person person) {
        return (Long) insert("ibatis.Person.insert", person);
    }
}
```

##HelloControl.java
```java
package com.springapp.mvc;
import dao.IAction;
import org.springframework.stereotype.Controller;
import org.springframework.ui.ModelMap;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.PathVariable;
import po.Person;
import java.io.*;
import java.sql.SQLException;
import javax.annotation.Resource;

@Controller
@RequestMapping("/")
public class HelloController {
	@Resource
	private Person p;
	@Resource
	private IAction action;
	@RequestMapping(method = RequestMethod.GET)
	public String printWelcome(ModelMap model) {
		model.addAttribute("message", "Hello world!");
		return "hello";
	}

	@RequestMapping(value="/add/{name}",method = RequestMethod.GET)
	public String addUser(@PathVariable String name,ModelMap model) throws IOException,SQLException {
		p.setName(name);
		p.setSex(2);
		action.insertPerson(p);
		model.addAttribute("message", "add person" + name);
		return "hello";
	}
}
```


  [1]:http://7u2qr4.com1.z0.glb.clouddn.com/blog_TimLine%E6%88%AA%E5%9B%BE20150206160918.png
  [2]: http://7u2qr4.com1.z0.glb.clouddn.com/blog_61b32fbb-1c8f-35ae-91cd-05dfd027b123.png