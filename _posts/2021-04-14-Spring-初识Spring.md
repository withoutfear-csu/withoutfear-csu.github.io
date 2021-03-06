---
layout: post
title: Spring-初识Spring
categories: Spring
description: 初识Spring,重点理解什么是IoC
keywords: Spring
---



 `Max Sheeran` 学 `Spring` 的第一天。主要了解什么是 `Spring`，什么是`IoC`。

## 什么是Spring

> **简介**

- Spring 是一个轻量级的开源框架
- Spring 的目的是为了解决业务逻辑层和其他各层的耦合问题
- Spring是一个IOC和AOP的容器框架
  - IOC:(DI) 控制反转(依赖注入)
  - AOP: 面向切面编程    - - - AOP是在IOC的基础上实现的
  - 容器：包含并管理应用对象的生命周期

> **相关链接**

**官网地址：**https://spring.io/projects/spring-framework#learn

**Spring5中文文档：**https://github.com/DocsHome/spring-docs/blob/master/SUMMARY.md

**压缩包地址：**https://repo.spring.io/

## Spring的模块划分

![Spring-Framework-Runtime](/images/posts/Spring/Spring-Runtime.png)

模块介绍：

- **Test**：Spring的单元测试模块
- **Core Container**：核心容器模块
- **AOP+Aspects**：面向切面编程模块
- Instrumentation：提供了class instrumentation支持和类加载器的实现来在特定的应用服务器上使用, 几乎不用
- Messaging：包括一系列的用来映射消息到方法的注解,几乎不用
- Data Access/Integration：数据的获取/整合模块，包括了JDBC,ORM,OXM,JMS和事务模块
- Web：提供面向web整合特性

## IoC

### 什么是IoC？

**官网说明**

> **IoC** is also known as dependency injection (**DI**). It is a process whereby objects define their dependencies (that is, the other objects they work with) only through constructor arguments, arguments to a factory method, or properties that are set on the object instance after it is constructed or returned from a factory method. The container then injects those dependencies when it creates the bean. This process is fundamentally the inverse (hence the name, **Inversion of Control**) of the bean itself controlling the instantiation or location of its dependencies by using direct construction of classes or a mechanism such as the Service Locator pattern.

**简单理解**

​	对象由Spring创建、管理和装配

对IoC的更详细理解将出现在文末的[思考与总结](#思考与总结)

### IoC 和 DI的关系

> **DI(依赖注入) 是IoC的具体实现**

​	2004年，Martin Fowlert提出：IOC到底是“哪些方面的控制被反转了呢？”

​	经过详细地分析和论证后，他得出了答案：“获得依赖对象的过程被反转了”。控制被反转之后，获得依赖对象的过程由**自身管理**变为了由**IOC容器主动注入**。于是，他给“控制反转”取了一个更合适的名字叫做“依赖注入（Dependency Injection）”。

​	他的这个答案，实际上给出了实现IOC的方法：注入。所谓依赖注入，就是由IOC容器在运行期间，动态地将某种依赖关系注入到对象之中。

​	所以，依赖注入(DI)和控制反转(IOC)是从不同的角度的描述的同一件事情，就是指通过引入IOC容器，利用依赖关系注入的方式，实现对象之间的解耦。

### 搭建 Spring - IoC 框架(代码演示)

#### 共有代码

> **项目结构**

<img src="/images/posts/Spring/proj-structure.png" alt="项目结构" style="zoom: 80%;" />

> **需要导入的jar包**

![](/images/posts/Spring/preparation.png)

> **创建接口**

`Tips`: 创建接口的原因见——[为什么使用接口](#为什么使用接口)

- dao.IUserDao

  ```java
  public interface IUserDao {
      void getUser();
  }
  ```

- service.IUserService

  ```java
  public interface IUserService {
      void getUser();
  }
  ```

> **dao 实现**

- dao.UserDaoMySQLImpl

  ```java
  public class UserDaoMySQLImpl implements IUserDao {
      @Override
      public void getUser() {
          System.out.println("查询MySQL数据库");
      }
  }
  ```

- dao.UserDaoOracleImpl

  ```java
  public class UserDaoOracleImpl implements IUserDao {
      @Override
      public void getUser() {
          System.out.println("查询Oracle数据库");
      }
  }
  ```


#### 不同代码

> **旧方法**

1. service.UserServiceImpl

   ```java
   public class UserServiceImpl implements IUserService {
       // 使用 旧方法
       // 如果 有很多类 使用了 UserDaoImpl这个类时， 当改动这个类，需要改动多个类
       
       //查询MySQL数据库时
       IUserDao dao = new UserDaoMySQLImpl();
       //查询Oracle数据库时
       // IUserDao dao = new UserDaoOracleImpl();
       
       @Override
       public void getUser() {
           dao.getUser();
       }
   }
   ```

2. 在 MyTest 中进行测试

   ```java
   public class MyTest {
       public static void main(String[] args) {
           // 使用 原始的方法
           IUserService service = new UserServiceImpl();
           service.getUser();
       }
   }
   ```

>**使用 IoC**

1. service.UserServiceImpl

   ```java
   public class UserServiceImpl implements IUserService {
       // 使用 IOC
       IUserDao dao;
       @Override
       public void getUser() {
           dao.getUser();
       }
   
       public IUserDao getDao() {
           return dao;
       }
   
       public void setDao(IUserDao dao) {
           this.dao = dao;
       }
   }
   ```

2. 添加 Spring 配置文件

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
   
       <bean class="dao.impl.UserDaoOracleImpl" id="userDaoOracle"/>
       <bean class="service.impl.UserServiceImpl" id="userService">
           <!-- 注入对象 -->
           <property name="dao" ref="userDaoOracle"/>
       </bean>
   </beans>
   ```

3. 在 MyTest 中进行测试

   ```java
   public class MyTest {
       public static void main(String[] args) {
           // 使用 IOC
           ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
           IUserService service = applicationContext.getBean(IUserService.class);
           service.getUser();
       }
   }
   ```

> **小结**

​	使用 **IoC** 相比于 **旧方法** 的主要区别：

- 旧方法中 **UserServiceImpl** 在初始化时需要主动去创建**UserDaoMySQLImpl**或者**UserDaoOracleImpl**，控制权在**UserServiceImpl**中。
- 使用 **IoC** 后，当**UserServiceImpl** 需要用到 **UserDaoMySQLImpl** 或者 **UserDaoOracleImpl** 时，IOC容器会主动创建一个对象注入到需要的地方。

### 为什么使用接口

> **不使用接口**

​		~~UserDaoImpl~~ dao=new ~~UserDaoImpl~~();

> **使用接口**

​		IUserDao dao=new ~~UserDaoImpl~~();



可见，使用接口能够一定程度上减轻耦合程度，要修改的部分就少了(当系统庞大时，效果对比会很明显)



### 依赖倒置原则(DIP)

> 依赖倒置原则（Dependence Inversion Principle）是程序要依赖于抽象接口，不要依赖于具体实现。简单的说就是要求对抽象进行编程，不要对实现进行编程，这样就降低了客户与实现模块间的耦合。

​		**IoC**是**DIP**的设计原理，**DI**(依赖注入) 是**IoC**的具体实现。

### IoC 的优点

1. 集中管理
2. 功能可复用(减少对象的创建和内存消耗)
3. 使得程序的整个体系结构可维护性、灵活性、扩展性变高
4. 解耦

## 思考与总结⭐

> 不使用 **IoC** 时

​	对象间的耦合关系会非常复杂，假设一个对象可以看作一个齿轮，那么就如下图所示

![](/images/posts/Spring/coupling.png)

> 引入 **IoC** 以后

![](/images/posts/Spring/decoupling.png)

​	由于引进了中间位置的“第三方”，也就是IOC容器，使得A、B、C、D这4个对象没有了耦合关系，齿轮之间的传动全部依靠“第三方”了，全部对象的控制权全部上缴给“第三方”IOC容器，所以，IOC容器成了整个系统的关键核心，它起到了一种类似“粘合剂”的作用，把系统中的所有对象粘合在一起发挥作用，如果没有这个“粘合剂”，对象与对象之间会彼此失去联系。这就是有人把IOC容器比喻成“粘合剂”的由来。

​	拿掉中间的大齿轮(IoC容器)后，再来看看

![](/images/posts/Spring/decoupling(2).png)

​	这时候，A、B、C、D这4个对象之间已经没有了耦合关系，彼此毫无联系，这样的话，当你在实现A的时候，根本无须再去考虑B、C和D了，对象之间的依赖关系已经降低到了最低程度

> 为什么 **IoC** 被称为 **IoC**?

​	软件系统在没有引入IOC容器之前，对象A依赖于对象B，那么对象A在初始化或者运行到某一点的时候，自己必须主动去创建对象B或者使用已经创建的对象B。无论是创建还是使用对象B，控制权都在自己手上。

​	由于IOC容器的加入，对象A与对象B之间失去了直接联系，所以，当对象A运行到需要对象B的时候，IOC容器会主动创建一个对象B注入到对象A需要的地方。

​	通过前后的对比，我们不难看出来：对象A获得依赖对象B的过程,由主动行为变为了被动行为，控制权颠倒过来了，这就是“控制反转”这个名称的由来。

## 参考文档

【1】DebugLZQ的博文：https://www.cnblogs.com/DebugLZQ/archive/2013/06/05/3107957.html

【2】Spring官网：https://spring.io