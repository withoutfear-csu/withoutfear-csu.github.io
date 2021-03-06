---
layout: post
title: Spring-IOC基于JavaConfig的配置方式
categories: Spring
description: 了解IoC基于JavaConfig的配置方式
keywords: Spring
---



`Max Sheeran` 学 `Spring` 的第四天。主要学习 `IoC` 基于 `JavaConfig` 的配置方式。**JavaConfig** 原来是 `Spring` 的一个子项目，它通过 `Java `类的方式提供 Bean 的 定义信息，在 Spring4 的版本， JavaConfig 已正式成为 Spring4 的核心功能 ，**SpringBoot**完全采用 **JavaConfig**的方式进行开发

## 与XML配置方式对比

​	在基于`Java`的容器配置中使用 **@Configuration**注解的形式替换了xml文件

|                  |                   XML配置方式                   |         JavaConfig配置方式         |
| :--------------: | :---------------------------------------------: | :--------------------------------: |
| 实例化容器的方式 |         ClassPathXmlApplicationContext          | AnnotationConfigApplicationContext |
|    设置包扫描    | &lt;context:component-scan base-package="?"&gt; | @ComponentScan(basePackages = "?") |
|     注册Bean     |           &lt;bean&gt; &lt;/bean&gt;            |               @Bean                |
| 引入外部资源文件 |     &lt;context:property-placeholder /&gt;      |          @PropertySource           |

## @Bean

​	添加在**方法**上，自动将方法的**返回值**作为Bean的**类型**，**方法名**作为Bean的**ID**。将类实例化后再交给IoC容器管理，注册为一个Bean。相比于XML的方式，可以自己干预Bean的实例化。

### 自动依赖外部Bean

​	直接在方法里面写上需要依赖的参数即可，:chestnut:中​注入的**role**就属于外部Bean

### 自动依赖内部Bean

​	直接调用方法即可，:chestnut:中注入的**user**就是属于内部Bean

### 举个:chestnut:

> 配置类**IoCJavaConfig**中

```java
@Bean
public DruidDataSource dataSource(Role role) {		// 在参数列表中加上Role，就可以依赖外部Bean
    DruidDataSource druidDataSource = new DruidDataSource();
    druidDataSource.setName(username);
    druidDataSource.setPassword(password);
    druidDataSource.setDriverClassName(driver);
    druidDataSource.setUrl(url);
    System.out.println(role.getName());				// 输出外部 Bean 的名称，证明依赖成功
    System.out.println(user());						// 打印user地址，和IoC容器中的user进行对比，以证明依赖成功
    return druidDataSource;
}
@Bean
public User user() {
    return new User();
}
```

> **测试中打印IoC容器中user的地址**

```java
@Test
public void test01() {
	AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(IoCJavaConfig.class);
    User user = ioc.getBean("user", User.class);
    System.out.println(user);
}

@Test
public void test02() {
    AnnotationConfigApplicationContext ioc = new AnnotationConfigApplicationContext(IoCJavaConfig.class);
    DruidDataSource dataSource = ioc.getBean("dataSource", DruidDataSource.class);
    System.out.println(dataSource);					// 以使用@Bean注释注册Bean成功
}
```

> **结果**

![](/images/posts/Spring/inject_inner_bean.png)

<img src="/images/posts/Spring/inject_bean.png" style="zoom:80%;" />

​	证明Bean注册成功，且成功依赖外部Bean和内部Bean。

### 注意点

- 若IoC容器中已存在同类型的同名Bean则会将其覆盖

## @Import

> **注解@Import的作用**

- 可以往**主配置类**中导入其他的配置类(可以引入多个配置类)

- 也可以将一个或多个类注册为**Bean**

- 导入**ImportSelector**实现类，可以注册多个**Bean**(<font color="red">getBean()不能根据名字获取，必须根据类型获取</font>>)

  ```java
  @Component
  public class MyImportSelector implements ImportSelector {
      @Override
      public String[] selectImports(AnnotationMetadata importingClassMetadata) {
          // 可以以字符串的形式返回多个Bean
          // 字符串必须是类的完整限定名，getBean不能根据名字获取，必须根据类型获取
          return new String[]{Person.class.getName(), "com.xuwei.beans.Wife"};
      }
  }
  ```

- 导入**ImportBeanDefinitionRegistrar**实现类，可以注册多个**BeanDefinition**(<font color="red">getBean()可以通过名字获取</font>>)

  ```java
  @Component
  public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
      @Override
      public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
          GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
          beanDefinition.setBeanClass(Person.class);
          registry.registerBeanDefinition("person", beanDefinition);
      }
  }
  ```

`Tips`: **ImportSelector**和**ImportBeanDefinitionRegistrar**对学习**SpringBoot**会有帮助

