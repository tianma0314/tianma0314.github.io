---
layout:     post
title:      Scala初探之类和对象
subtitle:   类、字段、方法
date:       2019-04-08
author:     walker
header-img: img/scala-1.jpg
catalog: true
tags:
    - Scala
    - Language
---

## 类、字段和方法

类定义里，不论字段还是方法，统称为类的成员。字段，不论是用val或是var定义，都是指向对象的变量；方法，用def定义，包含了可执行的代码。字段保留了对象的状态和数据，方法则是使用这些数据执行对象的运算工作。

当类被实例化的时候，运行时环境会在内存中预留一些位置保存对象的状态映像，即变量的内容，示例如下：

```scala
class CheckSumAccumulator {
  var sum = 0;
}

// 然后实例化两次
val csa = new CheckSumAccumulator
val asc = new CheckSumAccumulator
```

那么两个对象在内存的中映像如图所示：

![class-field-in-memory][/img/class-field-in-memory.jpg]
