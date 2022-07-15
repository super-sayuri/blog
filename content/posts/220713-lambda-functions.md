---
title: "JAVA常用的函数式编成函数"
date: 2022-07-13T16:51:15+08:00
tags: ["函数式编程", "lambda", "java"]
categories: ["编程"]
---

简单地介绍一下函数式编程的一些常用数据处理，虽然这里用的是 java，但是其他的函数式编程也大同小异。

```java
// 以下的 s 都是这个
Stream<String> s = Stream.of("h", "el", "llo");
```

## 遍历 

### foreach

遍历所有的元素，对它做想做的事情：

```java
// Stream s
s.foreach(
    ele -> {
        System.out.println(ele);
    }
);
```

### peek

对第一个对象进行操作。

## 映射

### map

A 是原集合， 通过计算 A 中的每个元素 a 都可以便成 B 集合中一个元素 b，这种 A -> B 的映射关系可以用 map 来实现。

```java
// Stream<String> s
String<Integer> si = s.map(
    ele -> ele.length();
) // 1, 2, 3
```

### flatmap

如果 A 中的元素 a 通过运算之后结果是个集合 A‘， flatMap 的作用就是输出所有 A’ 里元素展开的集合 B。

```java
// Stream<String> s
s.flatMap(
    ele -> Stream.of(ele.toCharArray())
); // 'h', 'e', 'l', 'l', 'l', 'o'
```

### mapMulti

和 flatMap 类似，但是自定义化更强，消耗也更少。

```java
// Stream<String> s
s.mapMulti(
    (ele, c) -> {
        c.accept(ele.toUpperCase());
        c.accept(ele.toLowerCase());
    }
); // "H", "h", "EL", "el", "LLO", "llo"
```

这三个方法都有 `XXmapToInt`, `XXmapToLong`, `XXmapToDouble` 的方法。

## 筛选

### filter

把所有 f(ele) 为 true 的弄成一个新集合返回。

```java
// Stream<String> s
s.filter(
    ele -> ele.length() > 1 && ele.length() < 3
) // "el"
```

### limit

保留前 n 个。

```java
s.limit(1); //  "h"
```

### skip

跳过前 n 个。

```java
s.skip(1); //  "el", "llo"
```

### takeWhile

一直拿直到后面的值为 false。

```java
s.takeWhile(
    ele -> ele.length() <= 2
); //  "h", "el"
```

### dropWhile

一直丢直到后面的值为 fales，拿剩下的所有。

```java
s.dropWhile(
    ele -> ele.length() <= 2
); //  "llo"
```

### distinct

去重。

## 排序与查找

### sorted

排序，值的注意的是最好都是加上 comparator。

```java
s.sorted(
    (a, b) -> a.compareTo(b)
); // "el", "h", "llo"
```

### allMatch, anyMatch, noneMatch

全部/任意一个/没有一个符合特定条件。

```java
s.allMatch(
    ele -> ele.length() > 1
); // false
```

### count

集合元素个数

### findAny, findFirst

返回任意一个/第一个元素，值的注意的是如果是空会返回一个 Option\<Null>。

```java
s.findFirst(); // Option("h")
```

### max, min

最大/小值

```java
s.max(
    (a, b) -> a.length() - b.length()
); // Option("llo")
```
## reduce

reduce 的意思是将集合 A 里面所有的元素 a 通过某种运算返回一个特定的值 x 。(x 和 a 是同类型的)

```java
s.reduce(
    "", // 初始值 
    (a, b) -> a + " " + b // 方法
); // "h el llo"
```

## 收集

### collect

将 stream 转变回collection (list, set, map ...)，这个是自己定义的，有方便的 `toArray` 和 `toList`。