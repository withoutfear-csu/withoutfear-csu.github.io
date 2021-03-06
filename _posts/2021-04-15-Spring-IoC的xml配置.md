---
layout: post
title: Spring-IoC的xml配置
categories: Spring
description: 了解IoC的xml配置使用
keywords: Spring, SSM, IoC
---



 `Max Sheeran` 学 `Spring` 的第二天。主要了解基于`IoC`容器 基于`xml`的元数据配置。实际开发中，没有基于纯xml配置开发的。但对于 `新手` 而言，基于xml配置的更容易理解和入门，因此有学习的必要。

## IoC容器

### 概述

​	**ApplicationContext**是**Spring IoC**容器实现的代表，它负责实例化，配置 和组装Bean。容器通过读取配置元数据获取有关实例化、配置和组装哪 些对象的说明 。配置元数据可以使用XML、Java注解或Java代码来呈 现。它允许你处理应用程序的对象与其他对象之间的互相依赖关系。

### 配置元数据

- 使用**xml**的配置 :star:
- 基于**注解**的配置
- 基于**Java**的配置

### 容器的使用

```java
	ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring-ioc.xml");
// 在容器实例化的时候，就会根据xml配置文件 从上往下(使用depends-on 可以改变该顺序)
// 加载所有的Bean(设置为懒加载的除外，可以通过lazy-init设置)
    System.out.println("Spring容器已加载");
    // 第一种 创建方式
//    User bean = applicationContext.getBean(User.class);
    // 第二种 创建方式
//    User user = (User) applicationContext.getBean("user");
    // 第三种 创建方式
    // 当一个类型被创建的次数≥2次时，只能使用该方法
    User user = applicationContext.getBean("user", User.class);
//    applicationContext.getBean(User.class);
```



## Bean

### Bean的命名

> **使用 alias 标签取别名**

```xml
<alias name="user" alias="xuwei"></alias>
```

> **使用 name 属性取别名，可以使用 space , ;作为分隔符**

```xml
<bean class="com.xuwei.beans.User" id="user" name="user2 user3,user4;user5">
    <description> description 是用来描述这个Bean 是干嘛用的</description>
</bean>
```

### Bean的实例化

​	[依赖注入](#依赖注入)中的Bean实例化方式都是默认使用构造器，由IoC容器帮我们管理的。如果想要自己干预Bean的实例化，就需要采用工厂方法。

- 使用**静态工厂方法**实例化

  - 直接在**Person**类中创建静态方法 **createPersonFactory**

  - ```xml
    <bean class="com.xuwei.beans.Person" id="person" factory-method="createPersonFactory"></bean>
    ```

- 使用**实例工厂方法**实例化

  - 另外创建一个类**personFactory** 实现**createPersonFactoryMethod**方法

  - ```xml
    <bean class="com.xuwei.beans.PersonFactory" id="personFactory"/>
    <bean class="com.xuwei.beans.Person" id="person"
          factory-bean="personFactory"
          factory-method="createPersonFactoryMethod">
    </bean>
    ```

这两种方法都是为了实现工厂设计模式的

### Bean的作用域

可以通过 `scope` 属性进行设置

> **Singleton**(单例)

​	只会被创建一次

> **Prototype**(原型)

​	可以多次创建

### Bean 的生命周期

> **生命周期回调**(三种方法)

- 实现接口 (InitializingBean 和 DisposableBean)

  - ```java
    // 需要将 IoC 容器关闭，才能观察到 Bean 被销毁
    // 1. 初始化方法： 实现接口： InitializingBean 重写 afterPropertiesSet方法初始化会自动调用的方法
    // 2. 销毁的方法： 实现接口： DisposableBean 重写 destroy 方法 销毁的时候自动调用方法
    Person person = applicationContext.getBean("person", Person.class);
    System.out.println(person);
    applicationContext.close();
    ```

- 基于配置自定义方法（在 Person 中定义）

  - ```xml
    <bean class="com.xuwei.beans.Person" id="person" init-method="initByConf" destroy-method="destroyByConf"></bean>
    ```

同时使用时，接口实现的方法会优先调用，如下图所示：

![order](/images/posts/Spring/order.png)

## 依赖

### 依赖注入

> **基于setter方法的依赖注入** 可以引入p命名空间简化

```xml
<!--基于setter方法的依赖注入
    注意：这里的 name 属性对应于 User 类的Set方法的名字
    例如：
        setIdABC -> name="idABC 即： SetXxx -> name="xxx"
-->
<bean class="com.xuwei.beans.User" id="user6">
    <property name="username" value="withoutfear"/>
    <property name="id" value="1"/>
    <property name="realname" value="Max Sheeran"/>
</bean>

 <!--使用p命名空间来简化 它不支持集合-->
<bean class="com.xuwei.beans.Wife" id="wife" p:age="18" p:name="迪丽热巴"></bean>
```

> **基于构造函数的依赖注入** 可以引入c命名空间简化

```xml
<!--基于构造函数的依赖注入
 1. 将会调用自定义构造函数来实例化对象，就不会调用默认的无参构造函数
 2. name是根据构造函数的参数名来的， 比如：User(String idxx) ‐> name="idxx"
 3. name属性可以省略 但是要注意参数的位置
 4. 如果非要把位置错开 可以使用 name 或者 index 或者 type
 5. index 是下标 从0开始
 6. type 在位置错开情况下只能在类型不一样的时候指定才有明显效果
-->
<bean class="cn.tulingxueyuan.beans.User" id="user3">
	<constructor‐arg name="username" value="withoutfear"></constructor‐arg>
	<constructor‐arg name="id_xushu" value="1"></constructor‐arg>
	<constructor‐arg name="realname" value="Max Sheeran"></constructor‐arg>
</bean>

<!-- 使用c命名空间来简化 它不支持集合 -->
<bean class="com.xuwei.beans.Wife" id="wife3" c:age="18" c:name="迪丽冷巴"></bean>
```

### 复杂属性注入的配置细节

```xml
<bean class="com.xuwei.beans.Person" id="person">
    <property name="id" value="1"/>
	<!--设置null值-->
    <property name="name">
        <null></null>
    </property>
    <!--设置空值-->
    <property name="gender" value=""></property>

    <property name="wife" ref="wife"/>
<!--
	当泛型是基本数据类型时 使用value
	当泛型是 bean 时 使用bean
-->
    <property name="hobbits">
        <list>
            <value>唱歌</value>
            <value>跳舞</value>
        </list>
    </property>
<!--如果Map中是bean则使用val-ref来引用-->
    <property name="courses">
        <map>
            <entry key="Java" value="100"></entry>
            <entry key="数据库" value="99"></entry>
        </map>
    </property>
</bean>

<!--外部bean，bean可复用，但需要指明id-->
<!--内部bean，bean不可复用，但无需指明id-->
<bean class="com.xuwei.beans.Wife" id="wife">
    <property name="name" value="迪丽热巴"/>
    <property name="age" value="18"/>
</bean>
```

### 自动注入⭐

​	在上文中提到的将 `wife` 注入 `person` 依然需要自己配置，有了自动注入以后，就不需要人为地去注入了，可以节省很多人力。

> **autowire**属性

- default：默认不使用自动注入

- byType：根据类型去自动匹配，当找到 ≥ 2个满足条件的bean时会报错

  - ```xml
    <bean class="com.xuwei.beans.Person" id="person" autowire="byType"></bean>
    <bean class="com.xuwei.beans.Wife" id="wife" p:name="迪丽温吧" p:age="22"></bean>
    ```

- byName：根据set方法后面的名称进行匹配

  - ```xml
    <bean class="com.xuwei.beans.Person" id="person" autowire="byName"></bean>
    <bean class="com.xuwei.beans.Wife" id="wife" p:name="迪丽温吧" p:age="22"></bean>
    ```

- constructor：需要加入特定的构造函数

  - 根据构造器去匹配(构造器中参数的名字) -- 必须**完全匹配**(包括参数名字和参数个数) 例如

    - Person(Wife wife, User user) 必须同时拥有 wife 和 user才能注入

  - 若没能匹配到，则按类型去匹配

  - 根据类型匹配时，若有多个bean满足条件，就会注入失败但不会报错。解决方法如下

    - 可以通过设置想要注入的bean的 `primary` 属性为true。
    - 也可以将不想要参与注入匹配的bean的 `autowire-candidate` 属性设置为false。

  - ```xml
    <bean class="com.xuwei.beans.Person" id="person" autowire="constructor"></bean>
    <bean class="com.xuwei.beans.Wife" id="wife" p:name="迪丽温吧" p:age="22"></bean>
    ```

## Spring创建第三方Bean对象

### 编写配置文件

```xml
<bean class="com.alibaba.druid.pool.DruidDataSource" id="druidDataSource">
    <property name="username" value="root"></property>
    <property name="password" value="123456789"></property>
    <property name="driverClassName" value="com.mysql.cj.jdbc.Driver"></property>
    <property name="url" 
              value="jdbc:mysql://localhost:3306/learn_spring5?
                     characterEncoding=utf8&amp;
                     useSSL=false&amp;
                     serverTimezone=UTC">
    </property>
</bean>
```

`Tips`: 这里使用了8.0的驱动和5.7的数据库，要添加时区等额外信息

### 引入外部配置文件

需要有 **db.properties** 文件存放数据库信息

```xml
<bean class="com.alibaba.druid.pool.DruidDataSource" id="druidDataSource">
    <property name="username" value="${jdbc.username}"></property>
    <property name="password" value="${jdbc.password}"></property>
    <property name="driverClassName" value="${jdbc.driver}"></property>
    <property name="url" value="${jdbc.url}"></property>
</bean>
<context:property-placeholder location="db.properties"></context:property-placeholder>
```

