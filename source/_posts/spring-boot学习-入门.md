---
title: spring boot学习-入门
date: 2018-04-13 18:36:10
tags: spring boot
keywords: spring boot
categories: spring boot
description: spring boot学习笔记
---
# 简介

Spring Boot其设计目的是用来简化 Spring 应用的初始搭建以及开发过程。Spring Boot 充分利用了 JavaConfig 的配置模式以及“约定优于配置”的理念，能够极大的简化基于 Spring MVC 的 Web 应用和 REST 服务开发。

spring boot 让编码，配置，部署，监控变得更加简单。

spring boot提供许多“开箱即用”的依赖模块，这些依赖模块均以spring-boot-starter-xx命名。常用模块有

* spring-boot-starter : 核心模块，包括自动配置支持、日志和YAML
* spring-boot-starter-web : 支持 Web 应用开发，包含 Tomcat 和 spring-mvc
* spring-boot-starter-aop : spring aop支持
* spring-boot-starter-logging ：使用 Spring Boot 默认的日志框架 Logback；
* spring-boot-starter-log4j ：添加 Log4j 的支持
* spring-boot-starter-jetty ：使用 Jetty 而不是默认的 Tomcat 作为应用服务器
* spring-boot-starter-jdbc ：支持使用 JDBC 访问数据库
* spring-boot-starter-redis ：支持使用 Redis
* spring-boot-starter-data-mongodb ：包含 spring-data-mongodb 来支持 MongoDB
* spring-boot-starter-test ： spring test支持
* spring-boot-starter-actuator ： 添加适用于生产环境的功能，如性能指标和监测等功能

# spring boot版本

<strong><font color="red">Spring boot依赖于Spring Framework,spring boot目前的最新release版本是2.0.1，该最新版本需要Spring Framework 5.0.5.RELEASE或更高版本。而Spring Framework 5.0 代码库运行于Java 8之上，所以Spring boot也要求 Java 8或Java 9</font></strong>
spring boot 2.0.1版本使用maven3.2或Grandle4提供构建支持。

spring boot 2.0.1版本支持Servlet3.1+容器，包括Tomcat8.5，jetty9.4和Undertow1.4。

spring boot 2.0.1版本的新特性，可以直接围观{% post_link Spring-5新特性 %}


# 快速搭建web项目

<strong>步骤1： 新建maven项目（略）</strong>
<strong>步骤2： pom文件中引入相应模块</strong>

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.milan</groupId>
	<artifactId>spring-boot-demo</artifactId>
	<version>1.0</version>
	<name>spring-boot-demo</name>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.1.RELEASE</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<java.version>1.8</java.version>
		<skipTests>true</skipTests>
	</properties>
	<dependencies>
		<!-- 核心模块，包括自动配置支持、日志和YAML -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<!-- Spring web模块 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>
	<build>
		<finalName>spring-boot-demo</finalName>
		<plugins>
			<plugin>
				<artifactId>maven-compiler-plugin</artifactId>
				<configuration>
					<source>${java.version}</source>
					<target>${java.version}</target>
				</configuration>
			</plugin>
		</plugins>
	</build>
</project>
```

<strong>步骤3：新建controller</strong>

```java
package com.milan.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class FirstController {
	
	@RequestMapping("/hello")
	@ResponseBody
	public String sayHello() {
		return "Hello World！";
	}

}
```

<strong>步骤4：新建启动类</strong>

```java
package com.milan;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class SpringBootDemoApplication {
	
	public static void main(String[] args) {
		SpringApplication.run(SpringBootDemoApplication.class, args);
	}

}
```

<strong>步骤5：直接运行启动类，访问http://localhost:8080/hello</strong>

# 其它

## 多模块项目

## 打包

### 打成jar包

### 打成war包

## 运行

### 后台运行

```shell
nohup java -jar yourjar.jar &
```

说明：nohup 命令的用途就是不挂断地运行命令

### 运行脚本

<strong>stop.sh</strong>

```shell
#!/bin/bash
PID=$(ps -ef | grep yourapp.jar | grep -v grep | awk '{ print $2 }')
if [ -z "$PID" ]
then
    echo Application is already stopped
else
    echo kill $PID
    kill $PID
fi
```

<strong>start.sh</strong>

```shell
#!/bin/bash
nohup java -jar yourapp.jar &
```

<strong>综合stop.sh和start.sh的运行脚本 run.sh</strong>

```shell
#!/bin/bash
echo stop application
source stop.sh
echo start application
source start.sh
```
