---
title: Spring 5新特性
date: 2018-04-13 21:33:23
tags: spring5
categories: Spring
description: spring 5新特性简单总结
---
Spring Framework 5的新特性，可以参考 [What's new in Spring Framework 5](https://www.ibm.com/developerworks/library/j-whats-new-in-spring-framework-5-theedom/) 


## JDK升级

核心的 Spring Framework 5.0已使用了Java 8，利用Java 8对如下点做了修订

* 基于Java 8的反射增强，Spring Framework中方法参数可更高效地访问；
* 接口提供基于Java 8的默认方法构建；
* lambdas表达式和lambdas函数式编程
* ... ...

## 响应式编程

响应式编程是关于非阻塞的，事件驱动的编程。Spring WebFlux是Spring 5的响应式式编程的核心，具体可以看[Spring WebFlux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-introduction)了解Spring WebFlux。

## HTTP/2支持

Spring Framework 5.0将提供专用的HTTP / 2特性支持，以及对JDK 9中预期的新HTTP客户端的支持。
[HTTP/2](https://www.ibm.com/developerworks/library/wa-http2-under-the-hood/index.html)

## Kotlin

Kotlin是JetBrains支持函数式编程的面向对象语言。它的主要优势之一是它提供了与Java的良好互操作性。

## Spring MVC支持lastest API

spring 5中的Model-View-Controll框架可与WebFlux以及Jackson 2.9,Protobuf 3.0的最新版本一起使用，且支持[Java EE 8 JSON Binding API](http://json-b.net/)

## 测试

Spring 5支持Junit 5（支持Java 8 streams & lambdas）,可以在单元测试中利用Java 8的函数式编程。

TestContext 框架

针对响应式编程模型， spring-test 现在还引入了支持 Spring WebFlux 的 WebTestClient 集成测试的支持，类似于 MockMvc，并不需要一个运行着的服务端。使用一个模拟的请求或者响应，WebTestClient使用模拟请求和响应以避免运行服务器，并且能够直接绑定到WebFlux服务器基础结构。
e.g.
  ```java
  WebTestClient testClient = WebTestClient
   .bindToServer()
   .baseUrl("http://localhost:8080")
   .build(); 
  ```

## 其它

Spring Framework 5改进了组件扫描和识别的方式，从而提高了大型项目的性能。组件扫描现在发生在编译时，且候选组件会添加到META-INF/spring.components文件中的索引文件中。

被javax包中注解标记的组件和被@Index标记的类或接口都将被添加到索引。

Spring的传统类路径扫描尚未被删除，但仍作为后备选项。