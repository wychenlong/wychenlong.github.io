---
layout:     post
title:      Lambda表达式
keywords:   Java, lambda
category:   java
description: java 8函数表达式学习
tags:		[java]
---

# Lambda表达式和函数式接口
## Lambda表达式
+ Lambda表达式（也叫做闭包）,将一个函数当作方法的参数（传递函数），或者说把代码当作数据.
+ 闭包
    + 一个含有自由变量的函数，外部环境持有内部函数所使用的自由变量，对内部函数形成“闭包”

+ jdk1.8之前实现闭包通过内部类+接口
    ```java
        new Comparator<User>() {
            @Override
            public int compare(User o1, User o2) {
                return o1.getAge().compareTo(o2.getAge());
            }
        })
    ```
+ 语法结构
    + () -> 表达式 
    + () -> { body }
    + 用括号包裹，并以逗号分割的参数列表
    + 箭头符号，->
    + 函数体(body)——包括至少一句表达式，或这一个表达式块


+ 案例一

```java
        Collections.sort(userList,(user1,user2) -> user1.getAge().compareTo(user2.getAge()));

        Collections.sort(userList,(user1,user2) -> {
            int i = user1.getAge().compareTo(user2.getAge());
            return i;
        });
```
+ 回顾内部类特性
    + 1、内部类可以用多个实例，每个实例都有自己的状态信息，并且与其他外围对象的信息相互独立。

    + 2、在单个外围类中，可以让多个内部类以不同的方式实现同一个接口，或者继承同一个类。

    + 3、创建内部类对象的时刻并不依赖于外围类对象的创建。

    + 4、内部类并没有令人迷惑的“is-a”关系，他就是一个独立的实体。

    + 5、内部类提供了更好的封装，除了该外围类，其他类都不能访问

+ 总结
    + 用来实现单一的行为，并且该行为要被传递给其他的代码
    + 当只需要一个功能接口的简单实例，而不需要类型，构造方法和一些相关的东西时

