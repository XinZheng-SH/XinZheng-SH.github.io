---
     layout:     post   				    
     title:      Spring整合myBatis
     subtitle:   封装后的CRUD功能
     date:       2020-06-19		
     author:     Xin 						
     header-img: img/post-black-bg.jpg 	
     catalog: true 						
     tags:								
         - Java 
         - Spring
         - myBatis
         
---

## myBatis

> 一个封装后的，操作数据库的框架

spring框架集成了myBatis的功能，可以将两者的功能融合在一起。本文使用的连接池为阿里的druid。

### pom.xml配置

在<dependencies>中加入以下内容

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.20</version>
</dependency>
<!-- mybatis依赖 -->
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.5</version>
</dependency>
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis-spring</artifactId>
    <version>2.0.5</version>
</dependency>
<!-- spring依赖 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>5.2.7.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.7.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>5.2.7.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>5.2.7.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-tx</artifactId>
    <version>5.2.7.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.2.7.RELEASE</version>
</dependency>

<!-- 阿里德鲁伊连接池 -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.22</version>
</dependency>
```

涉及Resource中xml的编译，在<resources>中加入以下内容，可根据个人项目自定义删改

*resources应在plugin后*

```xml
<resources>
    <resource>
        <directory>src/main/java</directory>
        <includes>
            <include>**/*.properties</include>
            <include>**/*.xml</include>
        </includes>
        <filtering>false</filtering>
    </resource>
    <resource>
        <directory>src/main/resources</directory>
        <includes>
            <include>**/*.xml</include>
            <include>**/*.properties</include>
            <include>*.xml</include>
        </includes>
        <filtering>false</filtering>
    </resource>
</resources>
```

如还涉及其他的依赖，可以在pom.xml中自行添加

### spring-myBatis配置文件(applicationContent.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/aop
                           http://www.springframework.org/schema/aop/spring-aop.xsd
                           http://www.springframework.org/schema/tx
                           http://www.springframework.org/schema/tx/spring-tx.xsd
                           http://www.springframework.org/schema/context
                           http://www.springframework.org/schema/context/spring-context.xsd">
    <context:property-placeholder location="classpath:jdbc.properties"/>

    <!--设置连接数据库的数据，druid会自动根据url加载对应的driver，不需再自行配置-->
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"
          init-method="init" destroy-method="close">
        <property name="url" value="${jdbc.url}"/>
        <property name="username" value="${jdbc.user}"/>
        <property name="password" value="${jdbc.password}"/>
        <property name="maxActive" value="${jdbc.maxActive}"/>
    </bean>
    
    <!--该bean等同于mybatis中提供的SqlSessionFactoryBean类，负责创建SqlSessionFactory-->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <!--这里连接的是myBatis自己的配置文件，可以不写留空-->
        <property name="configLocation" value="classpath:myBatis.xml"/>
    </bean>
    
    <!--
		创建dao对象，使用SqlSession的getMapper(UserDao.class)。
        MapperScannerConfigurer:在内部调用getMapper()生成每个dao接口的代理对象。
	-->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
        <!--
			指定包名，包名是dao接口所在的包名。
            MapperScannerConfigurer会扫描这个包中的所有接口，把所有接口都执行一次getMapper()方法，			得到每个接口的dao对象。
            创建好的dao对象放入到spring的容器中。
			如何得到容器中的dao对象？对象的value=首字母小写的类名。
			如接口类名:StudentDao，那么就可以使用getBean("studentDao")
        -->
        <property name="basePackage" value="top.zhengxin1024.dao"/>
    </bean>
    
    <!--自定义一个Service接口，选择运用部分dao中的方法。
		然后实例化一个类应用接口，类的代码在下方。-->
    <bean id="userService" class="top.zhengxin1024.service.impl.UserServiceImpl">
        <property name="userDao" ref="userDao"/>
    </bean>
</beans>
```

##### UserServiceImpl.java

```java
package top.zhengxin1024.service.impl;

import top.zhengxin1024.dao.UserDao;
import top.zhengxin1024.entity.User;
import top.zhengxin1024.service.UserService;

import java.util.List;

public class UserServiceImpl implements UserService {

    private UserDao userDao;

    //xml文件中配置好对应的dao，使用set注入
    public void setUserDao(UserDao userDao){
        this.userDao = userDao;
    }

    @Override
    public List<User> selectAllUsers() {
        return userDao.selectAllUsers();
    }

    @Override
    public int insertUser(User user) {
        return userDao.insertUser(user);
    }
}

```

如何调用对应的方法？

```java
@Test
public void testService(){
    String config = "applicationContent.xml";
    ApplicationContext ac = new ClassPathXmlApplicationContext(config);

    UserService dao = (UserService) ac.getBean("userService");

    System.out.println(dao.selectAllUsers().get(0));

}
```

## 总结

以上是最基础的，在spring框架中整合myBatis对数据库进行CRUD的方法。

1. pom.xml & applicationContent.xml & myBatis.xml等配置文件的正确配置，如出错会无法运行
2. 创建实体类、接口类、接口实现的xml文件
3. 在applicationContent.xml文件中创建dao对象、执行CRUD
4. 也可额外创建服务类、额外的接口等，将多个SQL语句综合绑定在一起，实现更复杂的服务
5. 事务处理等其他内容未纳入本文