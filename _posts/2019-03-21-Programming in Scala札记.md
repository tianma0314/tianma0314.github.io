---
layout:     post
title:      Scala入门
subtitle:   数组，列表，元祖，集，映射
date:       2019-03-21
author:     walker
header-img: img/scala-1.jpg
catalog: true
tags:
    - Scala
    - Language
---

## 使用列表

现有如下代码：

```scala
val twoThree = List(2, 3)
val oneTwoThree = 1 :: oneTwoThree
println(oneTwoThree)
```

上面的代码输出为：

```scala
List(1, 2, 3)
```

**注意：**

:: 是列表类的一个操作符，它表示的是将新元素组合在列表的最前端



> Scala规则：
> 
> - 如果方法使用操作符来标注，如：a * b，那么左操作数是调用者，可以改写成：（a）.+(b)
> - 如果方法以冒号结尾，这种情况下，方法是被右操作数调用，以上边代码为例，则可改写为：twoThree.::(1)