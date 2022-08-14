---
title: AOP基础
date: 2018-03-26 23:33:54
tags: Spring AOP
categories: Spring
keywords: AOP,Spring AOP
description: AOP即面向切面编程，在学习Spring AOP前，先了解一下AOP的一些基础概念（包括pointcut,advice,apsect等），并简单总结一下Java中常见的AOP实现，最后总结一些AOP的应用场景。ps:这里有一些是我看《Spring揭秘》的一些笔记总结，科普文,高手请绕开。
---


### AOP基础概念

AOP（Aspect Oriented Programming）即面向切面编程。以下介绍AOP中一些概念。

#### 连接点(Joinpoint)

系统运行前，AOP功能需要织入到OOP的功能模块中。要进行织入，就要知道在系统的哪些执行点进行织入操作，这些将要在其上进行织入操作的系统执行点即Joinpoint。

以下为常见的Joinpoint类型：

- 方法调用:在方法被调用时所处的程序执行点
- 方法执行:方法内部逻辑执行开始时的程序执行点
- 构造方法调用：调用构造方法进行初始化的时点
- 构造方法执行:构造方法内部执行的开始时点
- 字段设置：对象某个属性通过setter方法被设置或直接被设置的时点
- 字段获取：对象某个属性被访问的时点，可以通过getter方法也可以直接方法
- 异常处理执行:异常抛出后，对应的异常处理逻辑执行的时点
- 类初始化:类中某些今天类型或静态块的初始化时点

#### 切点(Pointcut)

Pointcut指定了系统中符合条件的一组Joinpoint。
Pointcut的表达方式有：

- 直接指定Jointpoint所在方法名称：只限于Joinpoit较少且较为简单的情况
- 正则表达式:大部分Java平台的AOP产品都支持这种形式的Pointcut表达式，
  e.g.Spring AOP、JBoss AOP
- 使用特定的Pointcut表述语言:功能强大，实现复杂。
  e.g.Spring2.0后支持AspectJ的Pointcut表述语言来来指定Pointcut

#### 通知(Advice)

Advice代表将会织入到Joinpoint的横切逻辑，根据它在Joinpoint位置执行时机的差异或完成功能的不同，可以分成以下形式：

- Before Advice
  指在Joinpoint指定位置之前执行的Advice类型。通常，它不会中断程序执行流程，且会先于方法执行而执行;若必要也可通过在Before Advice中抛出异常来中断当前程序流程。
- After Advice
  指在相应Joinpoint之后执行的Advice类型，它还细分为如下三种:
  a. After returning Advice:在当前Joinpoint处执行流程正常完成后，才执行的Advice；
  b. After throwing Advice:在当前Joinpoint执行过程中抛出异常后才会执行；
  c. After Advice:或称为Finally Advice，不管Joinpoint处执行流程正常还是抛出异常都会执行
- Around Advice
  对Joinpoint进行"包裹"，在Joinpoint之前和之后都指定相应的逻辑，甚至于中断或忽略Joinpoint处原来程序流程的执行。
  Filter功能就是Around Advice的一种体现，使用Around Advice可以完成“资源初始化”、"安全检查"之类横切系统的关注点。
              
#### 方面(Aspect)

指对系统中的横切关注点逻辑进行模块化封装的AOP概念实体。可以包含多个Pointcut及相关Advice定义。
事务管理就是一个横切关注点

#### 织入和织入器

织入是指将AOP集成到OOP，完成横切关注点逻辑到系统的最终织入过程的“工具”即织入器(Weaver)
例如：JBoss AOP采用自定义类加载器完成织入操作，该自定义类加载器就是织入器；Spring AOP使用一组类来完成最终的织入操作，ProxyFactory类则是Spring AOP最常用的织入器。

#### 目标对象(Target)

符合Pointcut所指定的条件，将在织入过程中被织入横切逻辑的对象称为目标对象
由于Spring AOP是使用运行时代理实现的，因此该对象将始终是代理对象。

### Java中AOP实现机制

根据织入的时机，分为3种：(1)编译期织入（AspectJ）;(2)类加载时织入（AspectJ 5+）;(3)运行时织入（Spring AOP）。
在Java平台上可以使用多种方式实现AOP，下面几种方式是最常用的：

#### 动态代理

JDK1.3后引入了动态代理机制，可以在运行期间，为相应的接口动态生成对应的代理对象。所以将横切关注点逻辑封装到动态代理的InvocationHandler中，在系统运行期间，根据横切关点需要织入的模块位置，将横切逻辑织入到相应的代理类中。

由于动态代理机制只针对接口有效，所以所有需要织入横切关注点逻辑的模块类都得实现相应的接口。
    
Spring AOP默认情况下采用这种机制实现AOP。

#### 动态字节码增强

Java虚拟机加载的class文件都是符合一定规范的，所以只要交给JVM运行的文件符合java class 规范，程序就能运行。通常class文件都是从java源代码文件使用javac编译器编译而成的，但只要符合java class规范，也可以使用ASM或CGLIB等java工具库，在程序运行期间，动态构建字节码的class文件。

在上述前提下，需要织入横切逻辑的模块在运行期间通过动态字节码增强技术为这些系统模块类生成对应的子类，并将横切逻辑加到这些子类中，让应用程序在执行期间使用这些动态生成的子类，到达将横切逻辑织入系统的目的。
    
动态字节码增强技术不受限于接口，即使模块类没有实现相应的接口，仍可对其进行扩展，但若要扩展的类及类中的实例方法等声明为 final的话，则无法对其进行子类化的扩展。
    
Spring AOP在无法采用动态代理机制进行AOP功能扩展时，会使用CGLIB库的动态字节码增强支持来实现AOP功能扩展。

#### java代码生成

在早期的EJB容器中使用最多，目前已退休。
EJB容器提供声明性事务支持，属于一种AOP功能模块实现，早期EJB容器采用Java代码生成技术。EJB容器根据部署描述符文件提供的织入信息，会为相应的功能模块类根据描述符所提供的信息生成对应的Java代码，然后通过部署工具或部署接口编译Java代码生成相应的Java类；之后，部署到EJB容器的功能模块类可以正常工作了。

#### 自定义类加载器

所有的class都要通过相应的类加载器加载到Java虚拟机之后才能运行。默认的类加载器会读取class字节码文件，然后按照class字节码规范，解析并加载这些class文件到虚拟机运行。
可以通过自定义类加载器，在class文件加载到虚拟机时，将横切逻辑织入class文件。自定义类加载器通过读取外部文件规定的织入规则和必要信息，在加载class文件期间可以将横切逻辑添加到系统模块类的现有逻辑中，然后将改动后的class交给JVM运行。
    
通过类加载器可以对大部分类及相应实例进行织入，功能强大；这种方式最大的问题是类加载器本身的使用。某些应用服务器会控制整个类加载体系，这种场景下使用可能产生一定的问题。
JBoss AOP是采用自定义类加载器的方式实现。

#### 扩展AOL
AOP也是一种理念，类似于OOP，实现AOP的语言称为AOL，常见的AOL有AspectJ,AspectC++... ...
通过扩展AOL实现任何AOP

### AOP的应用场景

AOP常见的应用场景有如下几种：

* 权限控制
* 缓存控制
* 事务控制
* 审计日志
* 性能监控
* 分布式追踪
* 异常处理
