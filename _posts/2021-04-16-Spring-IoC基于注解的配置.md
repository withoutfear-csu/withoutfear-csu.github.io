---
layout: post
title: Spring-IoC基于注解的配置
categories: Spring
description: 了解IoC基于注解的配置
keywords: Spring, SSM, IoC
---



`Max Sheeran` 学 `Spring` 的第三天。学习的是 Spring2. 5支持的基于 注解 的元数据配置 (xml+注解)，在SSM框架开发中使用。

## IoC容器

### 概述

​	**ApplicationContext**是**Spring IoC**容器实现的代表，它负责实例化，配置 和组装Bean。容器通过读取配置元数据获取有关实例化、配置和组装哪 些对象的说明 。配置元数据可以使用XML、Java注解或Java代码来呈 现。它允许你处理应用程序的对象与其他对象之间的互相依赖关系。

### 配置元数据

- 使用**xml**的配置 ​​
- 基于**注解**的配置:star:
- 基于**Java**的配置

## 使用注解方式注册Bean

### 四种注解

- **@Controller** 	标记控制层的类注册为Bean组件

- **@Service**           标记业务逻辑层的类注册为Bean组件

- **@Repository**    标记在数据访问层的类注册为Bean组件

- **@Component**  标记非三层的普通的类注册为Bean组件

  

  这些注解的本质都是**@Component**，不是非要每个层对应相应的注解。用**@Controller**举个:chestnut:：

  ![@Controller](/images/posts/Spring/@Controller.png)

但注解分层使用，有很多好处

- 增强可读性
- 利于Spring管理

`Tips`： 注解要标注在类上面，标注在接口上会被忽略

### 使用步骤

- 在Spring配置文件中设置包扫描

- 在对应的类上加上相应的注解 (会自动将类名的首字母小写并设置为Bean的id)

> **设置包扫描时的细节**

```xml
<context:component-scan base-package="com.xuwei" >
<!--<context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>-->
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

:one:**base-package**: 需要扫码的包

- 排除扫描 **context:exclude-filter**
- 包含扫描 **context:include-filter**
  - type属性
    1. annotation    根据注解的完整限定名设置排除\包含
    2. assignable     根据类的完整限定名设置排除\包含
    3. aspectj           根据切面表达式设置排除\包含
    4. regex              根据正则表达式设置排除\包含

:two:use-default-filters: 设置全扫描

- true：全扫描
- false：全不扫描

## 使用注解

## 使用注解实现自动装配:star:

### @Autowired

- 默认优先根据类型去匹配 (如果是接口，会匹配接口的实现类，父类同理)
- 如果匹配到多个类型
  1. 会按照名字匹配
  2. 若还是没匹配，则报错，解决方法：
     - 修改属性名称去对应 Bean 的名称
     - 采用**@Service(value = "userService")** 修改 Bean 的名称 其中 Value可以省略
     - 使用**@Qualifier(str)** 设置属性的名字为str(覆盖了)
     - 可以通过**@Primary** 设置其中一个Bean为主要的
     - 使用泛型作为自动注入限定符
- @Autowired可以定义在 **成员变量**、**方法** 上

`Tips`：也可以使用**@Resource**

**@Resource 和 @Autowired 的区别：**

- @Resource 依赖jdk  @Autowired 依赖Spring
- @Resource 优先以名称匹配 @Autowired 优先以类型匹配

## 缺陷

​	这种方式不能完全抛开xml文件，在配置第三方Bean的时候，还是会需要使用xml文件进行配置。