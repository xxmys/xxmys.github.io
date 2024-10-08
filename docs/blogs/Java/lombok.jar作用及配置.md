---
title: lombok.jar作用及配置
copyright: true
tags:
  - lombok
categories:
  - Java
permalink: /java/6cvvmxwy93
date: 2019-10-31 09:16:01
---

# lombok.jar 作用及配置

### 下载

1、下载 lombok.jar（一定要最新版）

下载地址：https://projectlombok.org/download

### 作用

java lombok 插件为我们写程序提供了许多的方便，主要是在面向对象的类的封装这一块，会为我们提供很多方便的接口，减少程序中冗余的代码。

1.@Data
@Data 注解在类上，会为类的所有属性自动生成 setter/getter、equals、canEqual、hashCode、toString 方法，如为 final 属性，则不会为该属性生成 setter 方法。

2.@Getter/@Setter
如果觉得@Data 太过残暴（因为@Data 集合了@ToString、@EqualsAndHashCode、@Getter/@Setter、@RequiredArgsConstructor 的所有特性）不够精细，可以使用@Getter/@Setter 注解，此注解在属性上，可以为相应的属性自动生成 Getter/Setter 方法。

3.@NonNull
该注解用在属性或构造器上，Lombok 会生成一个非空的声明，可用于校验参数，能帮助避免空指针。

4.@Cleanup
该注解能帮助我们自动调用 close()方法，很大的简化了代码。

5.@EqualsAndHashCode
默认情况下，会使用所有非静态（non-static）和非瞬态（non-transient）属性来生成 equals 和 hasCode，也能通过 exclude 注解来排除一些属性。

6.@ToString
类使用@ToString 注解，Lombok 会生成一个 toString()方法，默认情况下，会输出类名、所有属性（会按照属性定义顺序），用逗号来分割。

通过将 includeFieldNames 参数设为 true，就能明确的输出 toString()属性。这一点是不是有点绕口，通过代码来看会更清晰些。

7.@NoArgsConstructor, @RequiredArgsConstructor and @AllArgsConstructor
无参构造器、部分参数构造器、全参构造器。Lombok 没法实现多种参数构造器的重载。

### 配置

1、将 lombok.jar 包放置到 eclipse 安装目录下。

![](https://cdn.jsdelivr.net/gh/xxmys/image/img/202408090902986.png)

2、修改.ini 文件，增加以下内容：将目录更改为自己目录

```
-javaagent:D:\eclipse\eclipse\lombok.jar
```
