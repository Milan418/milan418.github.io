---
title: Spring AOP使用
date: 2018-03-27 16:52:38
tags: Spring AOP
categories: Spring
description: Spring AOP使用有两种形式，一是通过XML配置，二是通过注解；本文以应用或配置示例介绍AOP XML配置和AOP注解方式。
---

Spring AOP使用有两种形式，一是通过XML配置，二是通过注解。

# 基于XML配置

AOP配置，有如下三种方式:
- 在&lt;aop:config>使用&lt;aop:pointcut>声明切点；该切点可以被多个切面关联
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
	xmlns:aop="http://www.springframework.org/schema/aop"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
         http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
         http://www.springframework.org/schema/aop
         http://www.springframework.org/schema/aop/spring-aop.xsd">

	<bean id="loggerApsect" class="com.milan.base.LogAspect" />
	
	<aop:config>
		<!-- 声明切点pointcut，一个切点是可以被多个切面关联的-->
		<!-- expression表示满足expression的方法调用就会触发切面 -->
		<aop:pointcut id="logPointCut"
			expression="execution(* com.milan.service.ProductionService.*(..)) " />
			
		<!-- 声明切面，id代表这个切面的名字，ref就是切面方法所在的类，method指定切面方法的名字 -->
		<aop:aspect id="logAspect" ref="loggerApsect">
			<!-- 声明advice,其中pointcut-ref 将该切面同对应的切点关联，切点触发就执行指定方法-->
			<!-- advice有5种：before，after，after-returning ，after-throwing，around -->
			<!-- arg-names可以指定method中参数-->
			<aop:around pointcut-ref="loggerCutpoint" method="log" />
			<aop:after pointcut-ref="loggerCutpoint" method="log" />
		</aop:aspect>
	</aop:config>
</beans>
```
- 在 &lt;aop:aspect> 中使用 &lt;aop:pointcut>声明切点，该切点只被该切面使用
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
	xmlns:aop="http://www.springframework.org/schema/aop"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
         http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
         http://www.springframework.org/schema/aop
         http://www.springframework.org/schema/aop/spring-aop.xsd">

	<bean id="loggerApsect" class="com.milan.base.LogAspect" />

	<aop:config>
		<aop:aspect id="logAspect2" ref="loggerApsect">
			<aop:pointcut id="logPointCut"
				expression="execution(* com.milan.service.ProductionService.*(..)) " />
			<aop:after pointcut-ref="loggerCutpoint" method="log" />
		</aop:aspect>
	</aop:config>
</beans>
```

- 匿名切入点bean,只能被当前advice使用
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
	xmlns:aop="http://www.springframework.org/schema/aop"
	xsi:schemaLocation="http://www.springframework.org/schema/beans
         http://www.springframework.org/schema/beans/spring-beans-4.3.xsd
         http://www.springframework.org/schema/aop
         http://www.springframework.org/schema/aop/spring-aop.xsd">

	<bean id="loggerApsect" class="com.milan.base.LogAspect" />

	<aop:config>
		<aop:aspect id="logAspect2" ref="loggerApsect">
			<aop:after pointcut="execution(* com.milan.service.ProductionService.*(..))" method="log" />
		</aop:aspect>
	</aop:config>
</beans>
```

# @AspectJ注解方式

## 使用示例

Spring AOP中使用Aspect注解，要先在Spring配置文件中先配置支持@ApsectJ
```xml
<aop:aspectj-autoproxy/>
```
示例如下：
```java
/**
 * 日志输出注解
 * @author Milan
 *
 */
@Retention(RetentionPolicy.RUNTIME)  
@Target(ElementType.METHOD)  
public @interface Log {
	
	public enum LogType{ALL,PARAM,RESULT}

    public LogType value() default LogType.ALL;

}
/**
 * 日志切面
 * @author Milan
 */
@Aspect
@Component
public class LogAspect {

	private static final Logger logger = LoggerFactory.getLogger(LogAspect.class);

	/**
	 * 监控@Log注解了的方法
	 */
	@Pointcut("@annotation(com.milan.basic.annotation.Log)")
	private void pointCutMethod() {
	}

	/**
	 * 定制一个环绕通知,计算执行时间
	 * 注意：1.要返回Object,因为环绕的方法可能有返回值；2.要么处理异常要么抛出异常
	 * @param joinPoint
	 * @throws Throwable
	 */
	@Around("pointCutMethod()")
	public Object countTime(ProceedingJoinPoint joinPoint) throws Throwable {
		long begin = new Date().getTime();
		Object o = joinPoint.proceed();
		long end = new Date().getTime();
		logger.info("操作{}:执行时间{}ms", joinPoint.getTarget().getClass() + "." + joinPoint.getSignature().getName(),
				end - begin);
		return o;
	}
	@AfterReturning(pointcut="pointCutMethod()",returning="returnValue")
	public void log(JoinPoint point, Object returnValue) {
		try {
			Method method = ((MethodSignature)point.getSignature()).getMethod();
	        Log log = method.getAnnotation(Log.class);
	        Object[] args = point.getArgs();
	        String methodName = point.getSignature().getName();
	        if (log.value()== Log.LogType.ALL){
	        	logger.info("方法: "+methodName+" ,参数: "+JsonUtil.toJsonString(args)+"\n结果: "+returnValue);
	        }
	        if (log.value()== Log.LogType.PARAM) {
	        	logger.info("方法: "+methodName+" ,参数: "+JsonUtil.toJsonString(args));
			}
	        
	        if (log.value()== Log.LogType.RESULT){
	        	logger.info("方法: "+methodName+" ,结果: "+returnValue);
	        }
		} catch (Exception e) {
			logger.error("日志错误",e);
		}
	}
}
```

注解说明：

- @Aspect 用来表明该类是切面
- @Pointcut 用切面表达式来描述切点
- @Before 标识一个前置增强方法
- @Around 环绕增强，相当于MethodInterceptor
- @After final增强，不管是抛出异常或者正常退出都会执行
- @AfterReturning 后置增强，方法正常退出时执行
- @AfterThrowing 异常抛出增强，异常抛出时执行

## 切面表达式

切面表达式由以下三个部分组成：

- designators 指示器，描述通过何种方式匹配想要的方法或类
- Wildcards 通配符
    * \* 匹配任意字符
    * \+ 匹配指定类及其子类
    * .. 匹配任意数的子包或参数
- operators 运算符(包括&&,||,!),多个切点函数可以进行逻辑运算

以下详细说明designators,通常有以下几种匹配方式：

- 匹配方法 execution(<修饰符模式>?<返回类型模式><方法名模式>(<参数模式>)<异常模式>?)
  ?表示可选项，即修饰符和异常为可选配置，示例可参考Spring官方文档。

  ```java
    //匹配所有public修饰的方法
    @Pointcut("execution(public * * (..))")

    //匹配所有add开头的方法
    @Pointcut("execution(* add*(..))")

    //匹配所有UserService的所有方法
    @Pointcut("execution(* com.milan.service.UserService.*(..))")

    //匹配service包及其子包中以Service结尾的类下中所有方法
    @Pointcut("execution(* com.milan.service..*Service.(..))")

    //匹配add开头有3个参数且前两个参数为指定类型的方法
    @Pointcut("execution(* add*(String,java.util.List,*))")

    //匹配add开头至少有两个参数，且类型固定的方法
    @Pointcut("execution(* add*(String,int,..))")

    //匹配add(Object)方法
    @Pointcut("execution(* add(Object))")

    //匹配方法名为add,且参数为任意类的方法
    @Pointcut("execution(* add(Object+))")

    //匹配抛出IllegalAccessException的所有方法
    @Pointcut("execution(* *(..) throws java.lang.IllegalAccessEception)")
  ```
- 匹配注解
  匹配注解的有@annotation(),@within(),@target(),@args()。
  说明：在Spring Context环境下@within()和@target是没有区别的,非Spring context环境才有RetentionPolicy级别要求

  ```java
    //匹配@Log注解的方法
    @Pointcut("@annotation(com.milan.basic.annotation.Log)")

    //匹配@Log注解的类下所有方法，注意Log的RetentionPolicy级别为CLASS
    @Pointcut("@within(com.milan.basic.annotation.Log)")

    //匹配@Log注解的类下所有方法，注意Log的RetentionPolicy级别为RUNTIME
    @Pointcut("@target(com.milan.basic.annotation.Log)")

    //匹配传入参数类型被@Log标注的方法
    @Pointcut("@args(com.milan.basic.annotation.Log)")
  ```
- 匹配包或类，使用within()
  e.g.
  ```java
  //匹配UserService类的任意方法
  @Pointcut("within(com.milan.service.UserService)")

  //匹配service包下的所有方法
  @Pointcut("within(com.milan.service.*)")

  //匹配service包及其子包下的所有方法
  @Pointcut("within(com.milan.service..*)")
  ```
- 匹配对象,使用this(),target(),bean()
  e.g.
  ```java
  //匹配目标对象的Spring AOP代理对象中的方法,不支持通配符
  //匹配UserService的AOP代理对象中的方法
  @Pointcut("this(com.milan.service.UserService)")

  //目标类按类型匹配target中指定类型
  //匹配UserService的子类或实现类中的方法
  @Pointcut("target(com.milan.service.UserService)")

  //匹配spring容器中bean对象，通过bean的名称来匹配
  @Pointcut("bean(*Service)")
  ```
- 匹配参数,使用args()
  e.g.
  ```java
  //匹配server包中的类且第一个参数为long的方法
  @Pointcut("args(long,..) && within(com.milan.service.*)")
  ```
