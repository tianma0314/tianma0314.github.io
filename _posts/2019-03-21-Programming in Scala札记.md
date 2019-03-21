---
layout:     post
title:      Scala入门
subtitle:   Array, List, Tuple, Set, Map
date:       2019-03-21
author:     walker
header-img: img/scala-1.jpg
catalog: true
tags:
    - Scala
    - Language
---

### 使用列表

现有如下代码块-code01：

```scala
val twoThree = List(2, 3)
val oneTwoThree = 1 :: oneTwoThree
println(oneTwoThree)

// 输出为
List(1, 2, 3)
```

#### 特殊操作符

:: 是列表类的一个操作符，它表示的是将新元素组合在列表的最前端

> Scala规则：
> 
> - 如果方法使用操作符来标注，如：a * b，那么左操作数是调用者，可以改写成：（a）.+(b)
> - 如果方法以冒号结尾，这种情况下，方法是被右操作数调用，以上边code01代码为例，则可改写为：twoThree.::(1)
>
> Scala特例：
> 
> - 因为Nil是空列表的简写，所以可以使用 :: 操作符把各个元素都衔接起来，最后以Nil作为结尾来产生新列表，例如：
> ```scala
> val oneTwoThree = 1 :: 2 :: 3 :: Nil
> println(oneTwoThree)
> 
> // 输出为
> List(1, 2, 3)
> ```

::: 是叠加两个列表，从而产生一个新的列表，代码示例如下：

```scala
val oneTwo = List(1, 2)
val threeFour = List(3, 4)
val oneTwoThreeFour = oneTwo ::: threeFour
println(oneTwoThreeFour)

// 输出为
List(1, 2, 3, 4)
```

**思考：**

为什么List不支持append操作呢？因为append操作时间随着List变长而呈线性增长，而使用 :: 方法则消耗固定的时间，故若想获得一个可增长的列表，有如下两个方式：
1. 利用 :: 方法将元素前缀进列表，然后调用reverse方法
2. 使用ListBuffer，一种支持append操作的可变列表，完成后调用toList，获得不可变列表

### List的一些方法及其作用

方法名      |方法作用
:-----------|:------------------------------------------------
List()或Nil |空List
