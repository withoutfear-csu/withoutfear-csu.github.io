---
layout: post
title: Spring-初识AOP
categories: Spring
description: 了解AOP
keywords: Spring, AOP
---



`Max Sheeran` 学 `Spring` 的第五天。本文主要介绍什么是AOP，AOP的核心术语(切面、连接点、切入点、通知)，AOP的底层原理(JDK动态代理和CGLIB动态代理的区别)。

## 什么是AOP

### 简介

​	**AOP**：在不修改原有代码的情况下，通过将**公共功能代码**添加到**已有方法**中的指定位置，以达到增强主要业务功能的这种编程的方式。

### AOP和OOP比较

- **AOP**提供另一种思考程序结构的方式来补充**OOP**(面向对象编程)
- **OOP**中模块化的关键单元是**类(Class)**，在**AOP**中，则是**切面(Aspect)**

### AOP在Spring框架中的应用

- 提供声明式企业服务(**声明式事务管理**)。
- 让用户实现自定义切面，并通过**AOP**补充其对**OOP**的使用

### 英文文档

> > Aspect-oriented Programming (AOP) complements Object-oriented Programming (OOP) by providing another way of thinking about program structure. The key unit of modularity in OOP is the class, whereas in AOP the unit of modularity is the aspect. Aspects enable the modularization of concerns (such as transaction management) that cut across multiple types and objects.
>
> > **AOP is used in the Spring Framework to:**
>
> - Provide declarative enterprise services. The most important such service is **declarative transaction management**.
> - Let users implement custom aspects, complementing their use of OOP with AOP.

## AOP的核心术语

<img src="/images/posts/Spring/aop_terminology.png" style="zoom:80%;" />

### 切面

> **英文说明**
>
> > A modularization of a concern that cuts across multiple classes.

 	指关注点模块化，这个关注点可能会横切多个对象。

### 连接点

​	类中能被增强的方法，被称为连接点

### 切点

​	实际被真正增强的方法，被称为切点

### 通知

​	实际增强的逻辑部分称为通知。通知分为如下几类

- 前置通知（Before advice），在连接点之前运行但无法阻止执行流程前进到连接点的建议（除非它引发异常）。

- 后置返回通知（After returning advice），连接点正常完成后要运行的通知（如果方法返回而没有引发异常）。

- 后置异常通知（After throwing advice），如果方法因抛出异常而退出，则要运行的通知。

- 后置通知（总会执行）（After (finally) advice）:，无论连接点退出的方式如何（正常或异常返回），都应运行的通知。

- 环绕通知（Around Advice）:环绕连接点的通知，例如方法调用。

  > **英文文档**
  >
  > > This is the most powerful kind of advice. Around advice can perform custom behavior before and after the method invocation. It is also responsible for choosing whether to proceed to the join point or to shortcut the advised method execution by returning its own return value or throwing an exception.

## AOP的底层原理⭐

​	AOP的底层使用的是**动态代理**的设计模式

- JDK动态代理，需要被代理的类实现了接口
- CGLIB动态代理，不需要接口

### 简述什么是代理

​	代理模式就是在没有修改原有代码的基础上，对功能进行增强。

- **静态代理**
  - 需要为每一个被代理的类创建一个“代理类”
- **动态代理**
  - 所有被代理的类，都会经过一个公共执行方法

### 使用代码简单模拟

> **Demo包结构**

<img src="/images/posts/Spring/code_structure_aop.png" style="zoom: 67%;" />

`Tips`: 为了重点突出 AOP，在保持三层架构的基础上，使用尽量少的代码。

#### entity

> **User**

```java
public class User {
    // 用作保持三层架构，无意义
}
```

#### dao

> **IUserDao**

```java
public interface IUserDao {
    void method();
}
```

> **UserDaoImpl**

```java
@Repository
public class UserDaoImpl implements IUserDao {
    public void method() {
    }
}
```

#### service

> **IUserService**

```java
public interface IUserService {
    void method();
}
```

> **UserServiceImpl**

```java
@Service
public class UserServiceImpl implements IUserService {	// 将此处的接口去掉以后，可以验证CGLIB动态代理

    @Autowired
    IUserDao userDao;

    public void method() {
        userDao.method();
    }
}
```

#### controller

> UserController

```java
public class UserController {
    // 用作保持三层架构，无意义
}
```

#### aspect

> **LogAspect**

```java
@Aspect
@Component
public class LogAspect {
    @Before("execution(* com.xuwei.service.impl.*.*(..))")
    public void before() {
        System.out.println("前置通知");
    }
}

```

`Tips`: 这里的 **切面表达式**(execution 将在下一章介绍)

#### spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd">

    <context:component-scan base-package="com.xuwei"></context:component-scan>
<!--    开启AOP注解-->
    <aop:aspectj-autoproxy></aop:aspectj-autoproxy>
</beans>
```

#### 测试

```java
@Test
    public void test() {
        ClassPathXmlApplicationContext ioc = new ClassPathXmlApplicationContext("classpath:/spring.xml");
        // 没有使用AOP时 输出为 class com.xuwei.service.impl.UserServiceImpl
/* 
使用AOP时 
·若UserServiceImpl继承了接口，输出为 class com.sun.proxy.$Proxy18 说明使用了JDK的动态代理
·若UserServiceImpl没继承接口，输出为 class ... $$EnhancerBySpringCGLIB$$45a7c95e 说明使用CGLIB的动态代理	
*/

        // 删除接口后，需要使用具体类进行接收
//        UserServiceImpl bean = ioc.getBean(UserServiceImpl.class);
        /* 
        在继承了接口的情况下
        使用具体实现类，无法在容器中获取Bean
        	因为使用了动态代理，$Proxy继承了IUserService接口，而不是具体实现类UserServiceImpl
        	但是，使用名称可以正常获得
        */
//        IUserService bean = ioc.getBean(IUserService.class);
        IUserService bean = (IUserService) ioc.getBean("userServiceImpl");
        System.out.println(bean.getClass());
    }
```

