---
layout:     post
title:      Scala初探之集合对象
subtitle:   Array, List, Tuple, Set, Map
date:       2019-03-21
author:     walker
header-img: img/scala-1.jpg
catalog: true
tags:
    - Scala
    - Language
---

## 使用List

现有如下代码块-code01：

```scala
val twoThree = List(2, 3)
val oneTwoThree = 1 :: oneTwoThree
println(oneTwoThree)

// 输出为
List(1, 2, 3)
```

#### 特殊操作符

###### ::

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

###### :::

::: 是叠加两个列表，从而产生一个新的列表，代码-code02示例如下：

```scala
val oneTwo = List(1, 2)
val threeFour = List(3, 4)
val oneTwoThreeFour = oneTwo ::: threeFour
println(oneTwoThreeFour)

// 输出为
List(1, 2, 3, 4)
```

###### 思考

为什么List不支持append操作呢？因为append操作时间随着List变长而呈线性增长，而使用 :: 方法则消耗固定的时间，故若想获得一个可增长的列表，有如下两个方式：
1. 利用 :: 方法将元素前缀进列表，然后调用reverse方法。
2. 使用ListBuffer，一种支持append操作的可变列表，完成后调用toList，获得不可变列表。

#### List的一些特色方法及其作用

方法名      |方法作用
:-----------|:------------------------------------------------
List()或Nil |空List
val thrill = List("Will", "Bill", "Carl")|创建一个包含三个String类型元素的List，值分别为Will, Bill, Carl
thrill.init|返回thrill列表除最后一个元素以外的元素组成的列表（List("Will", "Bill")）
thrill.tail|返回thrill列表除第一个元素以外的元素组成的列表（List("Bill", "Carl")）
thrill.mkString(", ")|返回用列表元素组成的字符串（"Will, Bill, Carl"）

其他的像List.head，List.last这种顾名思义的方法就不详细介绍了。

## 使用Tuple

#### Tuple的特性

- 与列表一样，Tuple是不可变的
- 与列表不同，Tuple可以存储不同类型的元素
- Tuple实例化后，可以用点号、下划线和基于1的索引范围Tuple的元素

#### 创建和使用Tuple

代码-code03：
```scala
val pair = (99, "Lucas")
println(pair._1)
println(pair._2)

// 输出为
99
Lucas
```

Scala会推断Tuple类型为Tuple2[Int, String]，并赋值给pair。Tuple的类型取决于元素的数量和元素的类型，例如：(99, "Lucas")的类型为Tuple2[Int, String]；(99, 'u', "Fun", 'r')的类型为Tuple4[Int, Char, String, Char]

#### 访问Tuple的元素

Tuple无法像List那样去访问元素，如someList(0)，这是因为List的apply总是返回相同的类型，但是Tuple的元素类型不尽相同，_1和_2的类型可能不一致，因此两者的访问方法不一样。而且，_N的索引是基于1的

## 使用Set和Map

> 对于Set和Map来说，Scala提供了可变和不可变两种选择，但并非提供两种类型，而是通过继承的差别把可变性差异蕴含其中

#### Set

Scala的API中包含了Set的基本特质（trait），这个概念类似于Java中的接口（Interface），同时该基本特质下还有两个子特质，分别是可变的和不可变的，这三个特质拥有同样的简化名，但是他们所在的包不同，
而具体的Set类，如HashSet，则分别扩展（在Java中这叫做“实现”接口）了可变的和不可变的Set特质

###### 使用不可变Set

代码-code04：
```scala
var jetSet = Set("Boeing", "Airbus")
jetSet += "Lear"
println(jetSet.contains("Lesile"))
println(jetSet.getClass)

// 输出为
false
class scala.collection.immutable.Set$Set3
```

由code04代码可知，当不指明Set具体类型时，Scala会创建位于scala.collection.immutable包下的不可变Set实例，
在第二行中，当对不可变Set调用+=方法时，实际上是创建了一个添加新元素的新Set，并赋值给原Set，也就是说，第二行代码等同于：

```scala
jetSet = jetSet + "Lear"
```

###### 使用可变Set

由上面可知Scala默认创建不可变Set，若想创建可变Set，则需要引用可变Set，代码-code05示例如下：

```scala
import cala.collection.mutable.Set

val movieSet = Set("Mars", "Revange")
movieSet += "Piano"
println(movieSet)

// 输出为
Set("Mars", "Revange", "Piano")
```

由于movieSet是可变的Set故movieSet += "Piano"是真正的追加操作，等同于：

```scala
movieSet.+=("Piano")
```

#### Map

Map的继承结构与Set类似，也是有可变与不可变的两个特质（trait），并且若不指定哪种Map的话，默认创建不可变Map，这里不再详细介绍。

###### 使用可变Map

```scala
import scala.collection.mutable.Map

val treasureMap = Map[Int, String]()
treasureMap += (1 -> "A")
treasureMap += (2 -> "B")
treasureMap += (3 -> "C")

println(treasureMap(2))

// 输出为
B
```

## 总结

    


