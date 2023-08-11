---
title: Java List去重的几种方式
layout: post
published: true
categories:
  - Java
  - List
tags:
  - Java
index_img: /img/pages/Java.jpg
abbrlink: 40406fb8
date: 2023-08-08 21:06:47
---
在实际的业务开发中，经常会遇到List去重的场景，这里我将Java中常见的几种List去重方式记录一下；
<!-- more -->
常见的方式有以下几种：
#### 1. 使用 Set： 将列表转换为 Set，Set 不允许重复元素，然后再将 Set 转回列表。这会导致元素顺序发生改变
```java
List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 4, 5);
List<Integer> distinctNumbers = new ArrayList<>(new HashSet<>(numbers));
```
#### 2. 使用 Stream 的 distinct 方法： 使用 Stream API 提供的 distinct 方法，保持元素顺序
```java
List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 4, 5);
List<Integer> distinctNumbers = numbers.stream()
                                       .distinct()
                                       .collect(Collectors.toList());
```
#### 3. 使用 LinkedHashMap 保持顺序： 使用 Collectors.toMap 和 LinkedHashMap 以保持原始顺序
```java
List<Integer> numbers = Arrays.asList(1, 2, 2, 3, 3, 4, 5);
List<Integer> distinctNumbers = new ArrayList<>(new LinkedHashMap<>(
    numbers.stream()
           .collect(Collectors.toMap(
               Function.identity(),
               Function.identity(),
               (existing, replacement) -> existing
           ))
).values());
```
#### 4. 根据特定属性进行去重:如果您有自定义对象，并希望根据对象的某个属性进行去重，可以使用 Collectors.toMap
```java
List<Person> persons = Arrays.asList(
    new Person("Alice", "12345", 25),
    new Person("Bob", "12345", 30),
    new Person("Charlie", "67890", 28)
);

List<Person> distinctPersons = new ArrayList<>(new LinkedHashMap<>(
    persons.stream()
           .collect(Collectors.toMap(
               Person::getPhone,
               Function.identity(),
               (existing, replacement) -> existing
           ))
).values());
```
