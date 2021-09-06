---
layout: post
title: Java8-CompletableFuture
categories: Java8
description: 介绍CompletableFuture的使用
keywords: Java8, CompletableFuture
---

本文介绍为《Java8实战》的读书笔记，以几个自定义场景来说明CompletableFuture的使用场景和使用方式。



## 背景介绍

  仅仅使用Future中定义的方法的话不足以编写简洁的并发代码，比如以下需求：

- 将两个异步计算合并为一个，这两个计算之间相互独立，但第二个计算依赖于第一个计算的结果
- 等待Future集合中所有的任务完成、或仅等待最快的任务完成
- Future事件完成的时候会收到通知，而非简单的阻塞等待结果

为了轻松实现上述需求，引入了CompletableFuture，类似于Stream，他也实现了流水线和Lambda表达式的思想。

---



> **为方便说明，本文将在自定义的场景下展开介绍**

## 简单需求

​	实现一个价格比较器(**PriceHelper**)，它能获取各地商城(**Shop**)中某一特定商品的价格，来帮助消费者做出选择。每个商城都会向外提供一个自己的API来查询本商城的商品价格，以供外界调用。



### 商城(Shop)

#### Shop类定义

```java
public class Shop {
    private final Random random = new Random();

    private String name;

    public Shop(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
  
    // 价格查询同步API
		public double getPrice(String product) {
		    return calculatePrice(product);
		}
  
		// 价格查询异步API
		public Future<Double> getPriceAsync(String product) {
		    return CompletableFuture.supplyAsync(() -> calculatePrice(product));
		}
  
    // 计算商品价格
    private double calculatePrice(String product) {
        delay();
        return random.nextDouble() * product.charAt(0) + product.charAt(1);
    }

    // 模拟 I/O操作 以及 网络连接 等造成的延迟
    private static void delay() {
        try {
            Thread.sleep(1000L);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

}
```



> **商城提供的查询商品价格的API可能是异步的，也可能是同步的。**

##### 查询商品价格的同步API

​	使用同步API会使得调用方阻塞，直到结果返回才能去做其他计算。

```java
// 同步API 
public double getPrice(String product) {
    return calculatePrice(product);
}
```

##### 查询商品价格的异步API

​	异步API允许调用方在调用方法后立刻返回，去做一些其他计算，而非阻塞以等待结果。能够提高CPU的利用率，提高任务执行的效率。

```java
// 异步API
public Future<Double> getPriceAsync(String product) {
    return CompletableFuture.supplyAsync(() -> calculatePrice(product));
}
```

> **异步API的调用方式**

```java
Shop shop = new Shop("Shop");
long start = System.nanoTime();
// 调用异步API
Future<Double> futurePrice = shop.getPriceAsync("my favorite product");
long invocationTime = ((System.nanoTime() - start) / 1_000_000);
// 打印方法返回的时间
System.out.println("Invocation returned after " + invocationTime + " msecs");

// 可以做其他事情
//doSomethingElse()

try {
    // 获取异步API的返回结果
    double price = futurePrice.get();
    System.out.printf("Price is %.2f%n", price);
} catch (Exception e) {
    throw new RuntimeException(e);
}
long retrievalTime = ((System.nanoTime() - start) / 1_000_000);
// 打印方法完成的时间
System.out.println("Price returned after " + retrievalTime + " msecs");
```



> **作为调用方，不能苛求外界提供的API都是异步的。本章接下去讲在只能拿到同步的API的情况下，如何处理。**

### 价格比较器(PriceHelper)

​	假设商城只提供了同步的API，价格比较器需要实现一个“查询出某商品在不同商城所需价格“的功能(findPrice())

#### PriceHelper类定义

```java
public class PriceHelper {

    public List<Shop> shops;

    /**
     * 使用同步API
     */
    public List<String> findPrices(String product) {
        // 下面提供四种实现方案
    }
}
```



#### 测试findPrices性能

​	在介绍 `findPrices` 的四种实现时，需要涉及四种方案的比较，这里给出测试 `findPrices` 性能的方法：

```java
PriceHelper priceHelper = new PriceHelper();

priceHelper.shops = Arrays.asList(
        new Shop("ShopTest01"),
        new Shop("ShopTest02"),
        new Shop("ShopTest03"),
        new Shop("ShopTest04"),
        new Shop("ShopTest05"),
        new Shop("ShopTest06"),
        new Shop("ShopTest07"),
        new Shop("ShopTest08"),
        new Shop("ShopTest09"),
        new Shop("ShopTest10"),
        new Shop("ShopTest11"),
        new Shop("ShopTest12"));

long start = System.nanoTime();
System.out.println(priceHelper.findPrices("apple"));

long duration = (System.nanoTime() - start) / 1_000_000;
// 统计该方法所需的耗时
System.out.println("Done in " + duration + " msecs");
```



#### findPrices的四种实现

##### 串行阻塞

```java
public List<String> findPrices(String product) {
    // 方案一：
    return shops.stream()
            .map(shop -> String.format("%s price is %.2f",
                    shop.getName(), shop.getPrice(product)))
            .collect(toList());
            
}
```

**性能评估**

​	测试耗时大约为 **12s**



##### 并行流 

```java
public List<String> findPrices(String product) {
    // 方案二：仅做了细微的改动  ->  将 stream 换成了 parallelStream
    return shops.parallelStream()
            .map(shop -> String.format("%s price is %.2f",
                    shop.getName(), shop.getPrice(product)))
            .collect(toList());
}
```

**性能评估**

​	测试耗时大约为 **1s**（如若增加商城数量到**13**，耗时大约为 **2s**）



##### CompletableFuture+默认执行器

```java
public List<String> findPrices(String product) {
    // 方案三：
    List<CompletableFuture<String>> priceFutures =
            shops.stream()
                    .map(shop -> CompletableFuture.supplyAsync(
                            () -> String.format("%s price is %.2f",
                                        shop.getName(), shop.getPrice(product))
                    ))
                    .collect(toList());
    return priceFutures.stream()
            .map(CompletableFuture::join)
            .collect(toList());
}
```

**性能评估**

​	测试耗时大约为 **2s** （如若增加商城数量到**13**，耗时还是大约为 **2s**）



**细节说明**

> **1、为什么要将map拆成两段写，直接写在一起不行吗？**
>
> 答：由于流的延迟特性，写在一起，会使得最终的执行方式也是串行阻塞。



> **2、方式二和方式三效率不相伯仲的原因**
>
> 答：两种方式内部采用的是同样的通用线程池，线程数默认为Runtime.*getRuntime*().availableProcessors()的返回值(本次测试的机器返回12)。但使用CompletableFuture更具优势，允许对执行器进行配置。



**尚待解决的问题疑问**

> **为什么方式三: 12个商场时耗时2s , 23个时耗时3s；方式四: 12个商场耗时1s，24个商场，耗时2s。方式三每增加11个延时1s，方式四每增加12个延时1s。**



##### CompletableFuture+定制执行器

```java
public List<String> findPrices(String product) {
		// 方案四
		Executor executor = Executors.newFixedThreadPool(Math.min(shops.size(), 100),
		        r -> {
		            Thread t = new Thread(r);
		            t.setDaemon(true);
		            return t;
		        });
		List<CompletableFuture<String>> priceFutures =
		        shops.stream()
		                .map(shop -> CompletableFuture.supplyAsync(
		                        () -> String.format("%s price is %.2f", shop.getName(), shop.getPrice(product)),
		                        executor
		                ))
		                .collect(toList());
		
		return priceFutures.stream()
		        .map(CompletableFuture::join)
		        .collect(toList());
}
```

**性能评估**

​	采用这种方式后，即使有13个商店，依然能够在大约 **1s** 的时间内完成。



**细节说明**

> **为什么要使用守护线程？**
>
> > Java程序无法终止或退出一个正在运行中的线程，只要有用户线程运行，程序就不会终止。而守护线程则没有这一问题，程序退出自动回收。



**如何选择线程池的大小**

公式：**N<sub>threads</sub>** = **N<sub>CPU</sub>** * **U<sub>CPU</sub>** * (1 + **W**/**C**)

其中

- **N<sub>CPU</sub>**：处理器的核的数目，可以通过Runtime.*getRuntime*().availableProcessors()返回
- **U<sub>CPU</sub>**：期望的CPU利用率
- **W**/**C**：等待时间与计算时间的比率

实践中，要考虑到线程数不能超过任务数、线程数要有上限这些限制。



**并行——使用并行流还是使用CompletableFuture?**

​	对于没有**I/O**操作的计算密集型操作，使用Stream会更好一些，实现简单，并且对于计算密集型，没必要创建比处理器核数更多的线程。反之，采用CompletableFuture会更好一些。



> **下面为了介绍对多个异步任务的流水线操作，我们需要扩充一下需求背景**
>
> - 复杂需求A：介绍了**thenApply**和**thenCompose**的用法
>
> - 复杂需求B：介绍了**thenCombine**的用法



## 复杂需求A

​	现假设所有商城(**Shop**)使用的是一个集中式的折扣服务(**Discount**)，**getPrice**方法的返回格式现统一为: **[商场名]:[物品价格]:[折扣码]**，提供的依然只有同步API。由一个工具类**Quote**来解析。现在价格比较器(**PriceHelper**)需要返回所有商城中某一商品折后的价格。

### 商城(Shop)

#### 重构Shop.getPrice()

```java
public String getPrice(String product) {
    double price = calculatePrice(product);
    // 随机获取一个折扣码
    Discount.Code code = Discount.Code.values()[random.nextInt(Discount.Code.values().length)];
    return String.format("%s:%.2f:%s", name, price, code);
}
```



### 工具类Quote

​	Quote类的作用就是解析**Shop.getPrice()**方法的返回值，并包装成一个Quote类型。

#### Quote类定义

```java
public class Quote {
    private final String shopName;
    private final double price;
    private final Discount.Code discountCode;

    public Quote(String shopName, double price, Discount.Code code) {
        this.shopName = shopName;
        this.price = price;
        this.discountCode = code;
    }

    // 用于解析商城(Shop)的getPrice()的返回值
    public static Quote parse(String s) {
        String[] split = s.split(":");
        String shopName = split[0];
        double price = Double.parseDouble(split[1]);
        Discount.Code discountCode = Discount.Code.valueOf(split[2]);
        return new Quote(shopName, price, discountCode);
    }

    public String getShopName() {
        return shopName;
    }

    public double getPrice() {
        return price;
    }

    public Discount.Code getDiscountCode() {
        return discountCode;
    }
}
```



### 折扣服务(Discount)

​	**Discount**提供的打折API需要接收一个**Quote**类型的对象作为入参，来计算折扣。**Discount**属于远程服务，需要增加**1s**的模拟延迟。为方便说明，其向外界提供的API也假设只有同步API。

#### Discount类定义

```java
public class Discount {
    // 折扣代码，对应不同的折扣率
    public enum Code {
        NONE(0), SILVER(5), GOLD(10), PLATINUM(15), DIAMOND(20);
        private final int percentage;

        Code(int percentage) {
            this.percentage = percentage;
        }
    }
    // 提供给外部的折扣计算API
    public static String applyDiscount(Quote quote) {
        return quote.getShopName() + " price is " +
                Discount.apply(quote.getPrice(),
                        quote.getDiscountCode());
    }

    // 计算折后价格的操作
    private static double apply(double price, Code code) {
        delay();
        return (price * (100 - code.percentage) / 100);
    }
    // 1s 的模拟延迟
    private static void delay() {
        try {
            Thread.sleep(1000L);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```



### 价格比较器(PriceHelper)

​	通过**「简单需求」**的分析，我们已经知道了自定义CompletableFuture调度任务的执行器能够更充分地利用CPU资源，这里直接使用这一种方式。

#### 重构PriceHelper.findPrices()

```java
public List<String> findPrices(String product) {
    // 
    Executor executor = Executors.newFixedThreadPool(Math.min(shops.size(), 100),
            r -> {
                Thread t = new Thread(r);
                t.setDaemon(true);
                return t;
            });
    
    List<CompletableFuture<String>> priceFutures =
            shops.stream()
                    .map(shop -> CompletableFuture.supplyAsync(() -> shop.getPrice(product), executor))
                    // thenApply(): 
      						  .map(future -> future.thenApply(Quote::parse))
                    // thenCompose(): 上一个CompletableFuture的结果作为参数传入下一个
      							.map(future -> future.thenCompose(quote ->
                            CompletableFuture.supplyAsync(
                                    () -> Discount.applyDiscount(quote), executor)))
                    .collect(toList());
    return priceFutures.stream()
            .map(CompletableFuture::join)
            .collect(toList());
}
```

**thenApply和thenCompose的对比**

- thenApply: 其功能相当于将CompletableFuture<T>转换成CompletableFuture<U>，即转换的是范型中的类型
- thenCompose: 用于连接两个CompletableFuture，将上一个的结果作为后一个的入参，成为一个新的CompletableFuture

**性能评估**

​	测试耗时大约为2s



> **复杂需求A**中的两个任务是同步关系，必须拿到折扣码，才能计算折扣。下面**复杂需求B**将介绍没有同步关系的两个方法如何并行

## 复杂需求B

​	在「**复杂需求A**」的基础上，商城(**Shop**)提供商品原价API(**getOriginalPrice**)和商品描述API(**getIntroduction**)，要求价格比较器(**PriceHelper**)提供查询商品原价和相关描述的功能(**findOriginalPrices**)。

### 商城(Shop)

#### Shop.getOriginalPrice

```java
public double getOriginalPrice(String product) {
    return calculatePrice(product);
}
```

#### Shop.getIntroduction

```java
public String getIntroduction(String product) {
    delay();
    return product + "的介绍";
}
```

#### 价格比较器(PriceHelper)

#### PriceHelper.findOriginalPrices

```java
public List<String> findOriginalPrices(String product) {
    Executor executor = Executors.newFixedThreadPool(Math.min(2 * shops.size(), 100),
            r -> {
                Thread t = new Thread(r);
                t.setDaemon(true);
                return t;
            });
    List<CompletableFuture<String>> futures =
            shops.stream()
                .map(shop -> CompletableFuture.supplyAsync(() -> shop.getOriginalPrice(product), executor)
                // thenCombine: 将两个任务并行进行，结果合并
                .thenCombine(
                CompletableFuture.supplyAsync(() -> shop.getIntroduction(product), executor),
                (price, introduction) -> String.format("%s的价格为%.2f, 描述：%s", product, price, introduction)))
                .collect(toList());

    return futures.stream()
            .map(CompletableFuture::join)
            .collect(toList());
}
```

**性能评估**

​	这里getOriginalPrice()需要 **1s**, getIntroduction()也需要 **1s**。该测试总耗时为 **1s**，说明两个任务是并行进行的。



