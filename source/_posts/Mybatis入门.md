---
title: Mybatis入门
date: 2018-04-09 17:02:00
tags: mybatis
categories: mybatis
description: 之前工作用Hibernate，现在貌似都是更偏爱Mybatis，Mybatis入门简单能立马上手。本文就简单介绍一下Mybatis,以及和Hibernate的区别；最后在用spring boot+mybatis搭建一个demo。
---
## mybatis简介

mybatis官网是[http://www.mybatis.org/mybatis-3/](http://www.mybatis.org/mybatis-3/)。官网上Mybatis的Introduction如下：

>MyBatis is a first class persistence framework with support for custom SQL, stored procedures and advanced mappings. MyBatis eliminates almost all of the JDBC code and manual setting of parameters and retrieval of results. MyBatis can use simple XML or Annotations for configuration and map primitives, Map interfaces and Java POJOs (Plain Old Java Objects) to database records.

“咬文嚼字”翻译一下
>MyBatis是一个支持自定义SQL，存储过程和高级映射的持久性框架。 MyBatis消除了几乎所有的JDBC代码、参数设置和结果集的检索。 MyBatis可以使用简单的XML或注解来配置和映射原语，将Map接口和Java POJO 映射到数据库记录。

## Hibernate & mybatis

Hibernate是建立在若干POJO通过XML映射文件（或注解）提供的规则映射到数据库表上。Hibernate通过配置文件（或注解）就可以把数据库的数据直接映射到POJO上，我们可以通过操作POJP做操作数据库记录。Hibernate对JDBC的封装程度还是比较高的，不需要编写SQL语言，而采用HQL语言。

Hibernate优势：

* 配置了映射文件和数据库连接文件后，Hibernate就可以通过Session操作，非常容易，消除了JDBC带来的大量代码；
* 提供了级联、缓存、映射、一对多等功能。

劣势：

* 对多表关联和复杂SQL查询支持较差，需要自己写SQL&对结果进行组装；
* 不支持存储过程；
* 全表映射有诸多不便，例如更新是需要发送所有字段；
* HQL需要额外学习成本，性能较差，且SQL优化没法做。

MyBatis是一个半自动映射的框架，它需要手工匹配提供POJO，SQL和映射关系；MyBatis需要自己编写SQL，但是支持配置动态SQL。

## mybatis简单demo

入门的demo选用spring boot + mybatis（eclipse）。
步骤：
1. 新建maven项目；
2. pom中加入spring boot依赖 ，mysql依赖， mybatis依赖；

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.milan</groupId>
	<artifactId>mybatis-demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>

	<dependencies>
		<!-- 核心模块，包括自动配置支持、日志和YAML -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<!-- Spring web模块 -->
		<!-- 会增加web容器、springweb、springmvc、jackson-databind等相关的依赖 -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<!-- mysql依赖 -->
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
		<!-- mybatis依赖 -->
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>1.3.0</version>
		</dependency>
		<!-- 测试模块，包括JUnit、Hamcrest、Mockito -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>
	<build>
		<finalName>mybatis-demo</finalName>
		<!-- 设置jdk版本为1.8 -->
		<plugins>
			<plugin>
				<artifactId>maven-compiler-plugin</artifactId>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>
```

3. 在项目resource目录下添加application.properties文件,配置数据库信息 & mybatis

```text
#数据库配置
spring.datasource.url=jdbc\:mysql\://localhost\:3306/test?useUnicode\=true&characterEncoding\=utf-8&autoReconnect\=true
spring.datasource.username=username
spring.datasource.password=password
spring.datasource.driverClassName=com.mysql.jdbc.Driver

#mybatis配置
mybatis.type-aliases-package=com.milan.bean
```

4. 表设计略，表对应的bean如下

```java
package com.milan.bean;

public class User{
	private Long id;	
    private String name;
    private Integer age;
    //getter/setter略
}
```

5. mapper类设计，这里采用mybatis注解方式，sql就不放到专门的xml中了。

```java
package com.milan.mapper;

import java.util.List;

import org.apache.ibatis.annotations.Delete;
import org.apache.ibatis.annotations.Insert;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Result;
import org.apache.ibatis.annotations.ResultMap;
import org.apache.ibatis.annotations.Results;
import org.apache.ibatis.annotations.Select;
import org.apache.ibatis.annotations.Update;

import com.milan.bean.User;

@Mapper
public interface UserMapper {

	// 通过Parameter新增
    @Insert("INSERT INTO t_user(name, age) VALUES(#{name}, #{age})")
    int insertByParameter(@Param("name") String name, @Param("age") Integer age);

    // 通过Object新增
    @Insert("INSERT INTO t_user(name, age) VALUES(#{name}, #{age})")
    int insertByObject(User user);

    // Delete By Id
    @Delete("DELETE FROM t_user WHERE id =#{id}")
    void delete(Long id);

    // Update
    @Update("UPDATE t_user SET age=#{age} WHERE name=#{name}")
    void update(User user);

    // Find by Parameter
    @Select("SELECT * FROM t_user WHERE name = #{name}")
    List<User> findByName(@Param("name") String name);

    // 通过@Results，绑定返回值
    @Results(id="userMap",value={
        @Result(property = "name", column = "name"),
        @Result(property = "age", column = "age")
    })
    @Select("SELECT name, age FROM t_user")
    List<User> findAll();  
    
    //公用一个ResultMap
    @Select("SELECT name, age FROM t_user where id = #{id}")
    @ResultMap("userMap")
    User findUserById(@Param("id") Long id);
}
```

6. 集成测试不大会，所以我这直接写一个controller，通过链接访问测试，暂略；
7. spring boot启动类,直接允许该启动类后进行测试。

```java
package com.milan;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MybatisDemoApplication {
	public static void main(String[] args) {
		SpringApplication.run(MybatisDemoApplication.class, args);
	}
}
```
