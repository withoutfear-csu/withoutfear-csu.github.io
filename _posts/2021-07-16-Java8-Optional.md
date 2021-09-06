---
layout: post
title: Java8-Optional 
categories: Java8
description: 简单介绍Java8中的新特性——Optional
keywords: Java8, Optional 

---

本文介绍为《Java8实战》的读书笔记，简单介绍了Java8的新特性Optional。



## 什么是Optional

​	Optional 是一个可能包含空值，也可能不包含空值的容器对象。

## Optional相比于null的优势

​	**NullPointerException** 是Java开发中最常遭遇的异常，一般都是采用防御式检查的方式来减少 **NullPointerException** 异常的抛出，例如以下场景：

### 场景描述

​	获取某人（person） 所持有的车（car）的保险公司（insurance）的名字

### 使用null

```java
public class Person {
    private Car car;

    public Car getCar() {
        return car;
    }
}

public class Car {
    private Insurance insurance;

    public Insurance getInsurance() {
        return insurance;
    }
}

public class Insurance {
    private String name;

    public String getName() {
        return name;
    }
}
```

```java
public String getCarInsuranceName(Person person) {
    if (person != null) {
        Car car = person.getCar();
    if (car != null) {
            Insurance insurance = car.getInsurance();
            if (insurance != null) {
                return insurance.getName();
            }
        }
    }
    return "UnKnown";   
}
```

> **null 带来的问题**
>
> > 它让代码充斥着深度嵌套的null检查，可读性差。

## 引入Optional

​	引入了Optional之后，可以对上面的代码进行重构

```java
public class Person {
    // 代表对人而言，车是可选项
    private Optional<Car> car;

    public Optional<Car> getCar() {
        return car;
    }
}

public class Car {
    // 代表对车而言，保险是可选项
    private Optional<Insurance> insurance;

    public Optional<Insurance> getInsurance() {
        return insurance;
    }
}

public class Insurance {
    // 保险公司必有名字
    private String name;

    public String getName() {
        return name;
    }
}
```

```java
public String getCarInsuranceName(Optional<Person> person) {
    return person.flatMap(Person::getCar)
            .flatMap(Car::getInsurance)
            .map(Insurance::getName)
            .orElse("UnKnown");
}
```

> **引入Optional的好处**
>
> > **1）**减少了防御式检查造成的深层嵌套，代码可读性更好
> >
> > **2）**再报NullPointerException异常的时候，能够很快定位到业务逻辑错误（保险公司名称不能为空）
> >
> > **3）**调用方仅通过阅读函数签名，就能知道方法是否接受空值或返回空值

### Optional的使用方法

> **Optional的使用方法与Stream有多处类似**

#### 创建Optional对象

```java
// 空的Optional对象
Optional<Car> optCar = Optional.empty();
// 不接受空值的Optional
Optional<Car> optCar = Optional.of(car);
// 接受空值的Optional
Optional<Car> optCar = Optional.ofNullable(car);
```



#### 使用map提取、转换值

```java
Optional<Insurance> optInsurance = Optional.ofNullable(insurance); 
Optional<String> name = optInsurance.map(Insurance::getName);
```

**map的效果图：**

![map效果图](\images\posts\Java8\map效果图.png)

#### 使用flatMap链接对象

> 在重构的代码中，如果将 **flatMap(Car::getInsurance)** 替换为 **map(Car::getInsurance)**。得到的结果将是 **Optional<Optional<Car>>** 而不是 **Optional<Car>**

**flatma的效果图**

![flatmap效果图](\images\posts\Java8\flatmap效果图.png)

#### 使用filter筛选值

​	筛选出叫“CambridgeInsurance”的保险公司。

```java
Optional<Insurance> optInsurance = Optional.ofNullable(insurance); 
optInsurance.filter(insurance ->"CambridgeInsurance".equals(insurance.getName()))
            .ifPresent(x -> System.out.println("ok"));
```

#### 默认行为及解引用Optional

- get()：直接返回封装的变量，需确保变量存在。与null相差不大
- orElse(T Other)：若Optional对象不为空，则返回容器中的值。否则返回默认值 Other
- ifPresent(Consumer<? super T>)：若变量存在，则执行方法，否则什么也不做。

