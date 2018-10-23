---
layout:     post
title:      jdk8的optional深入详解
date:       2018-10-21
description:   jdk8的新特性 optional源码分析，深入详解，使用指南
toc: true
category: 
	- jdk8
tags:
    - jdk8
    - optional
    - 源码
---

# 为什么提供Optional类
在我们的开发中，NullPointerException可谓是随处可见，为了避免空指针异常，我们常常需要进行一些防御式的检查，所以在代码中常常可见if(obj != null)这样的判断，是非常讨厌的，碰到NPE，就需要对代码逐行检查是那个对象为空，带来大量不必要的精力耗费，抛出NPE异常不是用户操作的错误，而是开发人员的错误，开发人员使用了为空的对象造成的，应该避免，那么只能在每个方法中加入非空检查，阅读性和维护性都比较差

出现Optional主要是为了解决NPE(NullPointerExption)的问题
现在很多语言都对NPE做了一定程度上的规避，使用的都是Optional

如下面这些代码的手工非空检查
```java
public void addAddressToCustomer(Customer customer, Address newAddress){
    if(customer == null || newAddress == null){
        return;
    }
    
    if(customer.getAddress() == null){
        customer.setAddresses(new ArrayList());
    }
    
    customer.addAddress(newAddress);
}
```
另外还有一些开发人员喜欢通过非空检查来实现业务逻辑，空对象不应该用来决定系统的行为，它们是意外的Exception值，应当被看成是错误，而不是业务逻辑状态。

当我们一个方法返回List集合，应该总是返回一个空的List，而不是null，这样允许调用者能够遍历它而不必检查null，否则就抛出NPE。

但是如果我们根据标识键ID查询数据库，没有查询到，需要返回一个空对象怎么办，有人建议抛NPE。其实这不符合函数方法一进一出的原则，变成了一个函数方法两个返回值，一个是正常返回，一个出错Exception，函数式编程范式告诫我们不要轻易抛Exception。

幸好在JDK8中，java为了我们提供了一个Optional类，Optional类能让我们省掉繁琐的非空的判断。可以将Optional看成是需要我们使用某个T值的方法之间某种中间人或者协调者Mediator，而不是一个普通的对象的包装器。

如果你有一个值返回类型T, 你有一个方法需要使用这个值，那么你可以让Optional<T> 处于中间，确保它们之间交互进行，而不必要人工介入。

这样。协调者Optional<T>能够照顾T的值提供给你的方法做为输入参数，在这种情况下，如果T为空，可以确保不会出错，这样在T值为空时，也可以让一切都正常运行，你也可以让Optional<T>执行其他动作，如执行一段代码块等待，这样它就实际上是语言机制的很好补充。

# Optional的类源码
## Optional类的java doc
```
/**
 * A container object which may or may not contain a non-null value.
 * If a value is present, {@code isPresent()} will return {@code true} and
 * {@code get()} will return the value.
 *
 * <p>Additional methods that depend on the presence or absence of a contained
 * value are provided, such as {@link #orElse(java.lang.Object) orElse()}
 * (return a default value if value not present) and
 * {@link #ifPresent(java.util.function.Consumer) ifPresent()} (execute a block
 * of code if the value is present).
 *
 * <p>This is a <a href="../lang/doc-files/ValueBased.html">value-based</a>
 * class; use of identity-sensitive operations (including reference equality
 * ({@code ==}), identity hash code, or synchronization) on instances of
 * {@code Optional} may have unpredictable results and should be avoided.
 *
 * @since 1.8
 */
 一个容器对象，可能包含也可能不包含非null值。
 这是基于值的类，在Optional的实例上使用身份敏感操作(包括引用相等，hashcode或者同步)可能会产生不可预期的结果，应该避免使用。
不要使用==比较两个Optional对象
 
```
1： 类中的变量
```
EMPTY: 一个空的Optional实例，用来通过empty()重置被封装的对象值为空
value: 一个不是空的optional实例，被封装对象
```
2：类中public方法：

Optional中为了我们提供的方法。

| 方法      | 描述  
| :-------: | --------
| of        | 把指定的值封装为Optional对象，如果指定的值null，否则抛出NullPointerException
| empty     | 创建一个空的Optional对象 
| ofNullable| 把指定的值封装为Optional对象，如果指定的值null，则创建一个空的Optional对象
| get       | 如果创建的Optional中有值存在，则返回此值，否则抛出NoSuchElementException
| orElse    | 如果创建的Optional中有值存在，则返回此值，否则返回一个默认值
| orElseGet | 如果创建的Optional中有值存在，则返回此值，否则返回一个由Supplier接口生成的值
| orElseThrow| 如果创建的Optional中有值存在，返回此值，否则抛出一个由指定的Supplier接口生成的异常
| filter    | 如果创建的Optional中的值满足filter中的条件，则返回包含改值的optional对象，否则返回一个空的optional对象
| map       | 如果创建的optional的值存在，对改值执行提供的Function函数调用
| flatMap   | 如果创建的Optional的值存在，就对该值执行提供的Function函数调用，返回一个Optional对象，否则就返回一个空的Optional对象
| isPresent | 如果创建的Optional中的值存在，返回true，否则返回false
| ifPresent | 如果创建的Optional中的值存在，则执行该方法的调用，否则什么也不做

详细说明：

**of**
```java
// 创建一个值为张三的String类型的Optional
Optional<String> opt = Optional.of("张三");
// 如果我们用of方法创建Optionald对象时，所传入的值为null，则抛出NPE
Optional<String> opt = Optional.of(null);
```
**ofNullable**
```java
// 为指定的值创建Optional对象，不管所传入的值为null，不为null，所创建的时候都不会报错
Optional<String> opt = Optional.ofNullable(null);
Optional<String> opt = Optional.ofNullable("张三");
```
**empty**
```java
// 创建一个空的String类型的Optional对象
Optional<String> opt = Optional.empty();
```
**get**

如果们创建的Optional对象中有值则返回此值，如果没有值存在，则会抛出NoSuchElementException异常。
```java
OPtional<Sting> opt = Optional.ofNullable("张三");
System.out.println(opt.get()); // 张三

opt = Optional.empty();
System.out.println(opt.get()); // 抛异常
```
**orElse**

如果创建的Optional中有值存在，则返回此值，否则返回一个默认值
```java
Optional<String> opt = Optional.of("张三");
System.out.println(opt.orElse("zhangsan"));

opt = Optional.empty();
System.out.println(opt.orElse("lisi"));
```
**orElseGet**

如果创建的Optional中有值存在，这返回此值，否则返回一个有Supplier接口生成的值
```java
Optional<Sting> opt = Optional.of("zhangsan");
System.out.println(opt.orElseGet(() -> "lisi"));

opt = Optional.empty();
System.out.println(opt.orElseGet(() -> "orElseGet"));
```
**orElseThrow**

如果创建的Optional中有值存在，则返回此值，否则抛出由指定Supplier接口生成的异常
```java
Optional<String> opt = Optional.of("张三");
System.out.println(opt.orElseThrow(CustomException::new));

opt = Optional.empty();
System.out.println(opt.orElseThrow(CustomException::new));

private static class CustomException extends RuntimeException{
    private static final long serialVersionUID = 1;
    
    public CustomException(){
        this("自定义异常");
    }
    
    public CustomException(String msg){
        super(msg);
    }
}
```
**filter**

如果创建的OPtional中值满足filter的条件，则返回该值的Optional对象，否则返回一个空的Optional对象
```java
Optional<String> opt = Optional.of("zhangsan");
System.out.println(opt.filter(s -> s.length() > 5).orElse("wangwu"));

opt = Optional.empty();
System.out.println(opt.filter(s -> s.length() > 5).orElse("lisi"));
```
> 注意Optional中的filter方法和Stream中的filter方法是有点不一样的，Stream中的filter方法是对一堆元素进行过滤，而Optional中的filter方法只是对一个元素进行过滤，可以把Optional看成是最多只包含一个元素的Stream。

**map**

如果创建的Optional中的值存在，对该值执行Function函数的调用
```java
// map方法执行传入的lambda表达式对Optional实例的值进行修改，修改后的返回值仍然是一个Optional对象
Optional<String> opt = Optional.of("zhangsan");
System.out.println(opt.map(s -> s.toUpperCase()).orElse("error"));

opt = Optioanl.empty();
System.out.println(opt.map(s -> s.toUpperCase()).orElse("error"));
```
**flatMap**

如果创建的Optional中的值存在，就对该值执行提供的Function函数调用，返回一个Optional类型的值，否则就返回一个空的Optional对象，faltMap与map(Function)方法类似，区别在于faltMap中的mapper返回的值必须是Optional, map方法的mapping函数返回值可以是任意类型T，调用结束时，flatMap不会对结果用Optional封装。
```java
// map方法中的lambda表达式返回值可以是任意类型，[在map函数返回之前会包装为Optional
// 但flatMap方法中的lambda表达式返回值必须是Optional实例
Optional<String> opt = Optional.of("zhangsan");
System.out.println(opt.flatMap(s -> Optional.of("lisi")).orElse("error"));

opt = Optional.empty();
System.out.println(opt.flatMap(s -> Optional.empty)).orElse("error"));

```
**ifPresent**
```java
// ifPresent方法的参数是一个Customer的实现类，Customer类包含一个抽象方法，该抽象方法对传入的值进行处理，只是没有返回值。
Optional<String> opt = Optional.of("zhangsan");
opt.ifPresent(s -> System.out.println("我被处理了...." + s));
```

# 总结
- Optional类就是对对象进行封装用Optional中的方法更优雅的处理NPE的情况。
- 一般用法：在方法返回的时候通过of或者ofNullable方法转换成相对应对象，在调用的时候通过Optional提供的方法去做边界处理。常用作参数返回值
- 特别是和Stream结合的时候。需要一个默认值或者指定的异常的时候非常需要这种优雅的方式去处理。
- 提供的map。faltMap。filter主要用来更好的使用，特别是faltMap避免出现Optional<Optional<T>这种类型。