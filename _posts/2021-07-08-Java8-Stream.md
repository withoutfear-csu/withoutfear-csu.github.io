---
layout: post
title: Java8-Stream流式编程
categories: Java8
description: 介绍了Java8中的新特新——流式编程
keywords: Java8, Stream

---



这篇文章为《Java8 实战》的阅读笔记，简单地介绍了Java8的新特性——Stream流式编程。

## 流与集合

> **集合对很多Java程序来说是必要的。但集合的操作远谈不上完美，流的出现弥补了这一不足。**

​	很多业务逻辑都涉及到排序、去重、分组等类似于数据库的操作。在Java8之前，应对这一情况通常需要**临时编写**一个实现，而引入Stream之后允许程序以声明性方式来处理，举一个直观的例子(Stream使用的细节将在[**「流的使用」**](#流的使用)中介绍)：

### 需求描述

从菜单(menu)上选出选出卡路里(Calories)小于400的菜肴(Dish)，按卡路里从低到高列出菜肴的名字(name) 形成一个低卡路里菜单(lowCaloricDishes)。

- **使用 临时编写 的实现**

  ```java
  List<Dish> lowCaloricDishes = new ArrayList<>();
  // 筛选元素
  for(Dish d: menu){
  		if(d.getCalories() < 400){
  			lowCaloricDishes.add(d); 
   		}
   }
  // 按卡路里排序
  Collections.sort(lowCaloricDishes, new Comparator<Dish>() { 
  	public int compare(Dish d1, Dish d2){
  	  return Integer.compare(d1.getCalories(), d2.getCalories());
   	}
  }); 
  // 输出结果
  List<String> lowCaloricDishesName = new ArrayList<>();
  for(Dish d: lowCaloricDishes){
      lowCaloricDishesName.add(d.getName());
  }
  ```

- **使用 Stream API 的实现**

  ```java
  import static java.util.Comparator.comparing;
  import static java.util.stream.Collectors.toList;
  List<String> lowCaloricDishesName =
                 menu.stream()
                 .filter(d -> d.getCalories()<400)				// 筛选元素
                 .sort(comparing(Dishes::getCalories))		// 按卡路里排序
                 .map(Dish::getName)											// 映射菜名
                 .collect(toList());											// 结束输出结果
  ```

  > 想要利用并行架构，只需要将下面的 **stream** 替换为 **parallelStream 即可**



### 显而易见的好处

- 声明性，直接说明想要完成什么，更简洁、易读
- 可复合，多个操作可以链接起来就像流水线一样，更灵活
- 可并行，不需要自己实现并行处理，性能更好

Stream之所以拥有这些好处，是因为使用了内部迭代来代替外部迭代，为程序员设计者节约了宝贵的时间。



### 流与集合的差异

​    流与集合的主要差异体现在“**何时计算**”以及“**数据遍历”上**。

**何时计算：**集合中的所有元素都需要**先计算**出来后，才能添加到集合中去。而流中的元素都是**按需计算**的，因此有无限流的存在，但没有无限集合的存在。(类似于单例模式中“饿汉式”和“懒汉式”的差异)

**数据遍历：**流和迭代器类似同一个流只能遍历一次(再次遍历需要从原始数据流中再获取一份)，而对于同一个集合的话，可以多次遍历。

---



## 流的使用

​	流的使用，一般涉及 **构建流**、**中间操作链**、**终端操作** 三方面。

### 构建流

#### 对象流

​	可以使用Arrays.stream()、.stream()等方法进行创建。

#### 数值流

​	在单纯的数值计算场景下，如果使用对象流进行操作，那么就会涉及拆箱的成本(如：Integer->int, Long->long)。这种场景下使用原始类型的数据流会更好。

- IntStream
- LongStream
- DoubleStream

### 流的操作

​	流的操作可分为 **中间操作** 和 **终端操作**，我认为两者的区分点在于操作的返回值，中间操作返回的仍是一个流，还可以进行流操作；而终端操作返回的不是流，不能再进行流相关的操作。

#### 中间操作

**筛选**

- filter: 筛选出满足条件的数据

- distinct: 对数据进行去重
- limit: 取前 N 个数据（和 skip 互补）
- skip: 跳过 N 个数据，取后面的数据（和 limit 互补）



**映射**（映射与转换类似，但转换是 修改，映射则是 新建）

- map: 直接映射，如：

  **需求**：在菜单(menu)中提取菜肴的名称(Dish::getName())

  ```java
  List<String> dishNames = menu.stream()
                                   .map(Dish::getName)
                                   .collect(toList());
  ```

- flatmap: 扁平化映射，将多个流进行合并，如：

  **需求**：给一个字符串数组，问其中用到了哪些字母。如：输入：["abc", "bcd", "cde"]，输出：["a", "b", "c", "d", "e"]。

  ```java
  List<String> uniqueCharacters =
      words.stream()
           .map(w -> w.split(""))
           .flatMap(Arrays::stream)
           .distinct()
           .collect(Collectors.toList());
  ```

  > **问：flatmap和map有什么区别？为什么上述代码中第一次用map，第二次用flatmap**
  >
  > > ​	下图给出了解释，当使用两次map的时候，得到的是Stream<Stream<String>>，flatmap得到的则是Stream<String>。可简单认为flatmap能抵消一层Stream

- **图解（左分支是两次都使用map的结果，右分支是第一次用map，第二次用flatmap的结果）**

  ![map和flatmap对比](\images\posts\Java8\map和flatmap对比.jpg)



#### 终端操作



**匹配**

返回 **Boolean** 型变量 

- allMatch：检查是否“所有元素与条件匹配”

- anyMatch：检查是否“至少存在一个元素与条件匹配”
- noneMatch：检查是否“没有元素与条件匹配”



**查找**

返回 **Optional** 型变量

- findFirst：找到第一个符合条件的元素
- findAny：找到任意一个符合条件的元素

> **问：findFirst和findAny在并行场景下的不同**
>
> > 答：为了使用并行，使用**findFirst**的时候，对并行有很大的限制。
> >
> > 但**findAny**就不一样了，对位置没有要求，所以可以使用并行。



**归约**

- collect
- reduce



