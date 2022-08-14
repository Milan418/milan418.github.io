---
title: Spring AOP实现机制
date: 2018-03-28 19:22:34
tags: Spring AOP
categories: Spring
description: Spring AOP是运行时织入，运行时织入通过静态代理或动态代理实现，Spring AOP采用动态代理。而动态代理两种实现方式：基于接口的代理实现和基于继承的代理实现。本文对基于接口的代理实现-JDK动态代理和基于继承的代理实现-cglib代理，进行简单的使用示例说明。
---

## 代理模式

代理模式是常用的java设计模式，他的特征是代理类与委托类有同样的接口，代理类主要负责为委托类预处理消息、过滤消息、把消息转发给委托类，以及事后处理消息等。代理类与委托类之间通常会存在关联关系，一个代理类的对象与一个委托类的对象关联，代理类的对象本身并不真正实现服务，而是通过调用委托类的对象的相关方法，来提供特定的服务。

## 静态代理

静态代理:由程序员创建或特定工具自动生成源代码，再对其编译。在程序运行前，代理类的.class文件就已经存在了。
e.g.
```java
/**
 * 定义一个账户接口 
 */ 
public interface Count {  
      public void query(); // 查看账户方法  
      public void update(); // 修改账户方法
} 

/** 
 * 委托类(包含业务逻辑)  
 */ 
public class CountImpl implements Count {
   @Override    
   public void query() {
       System.out.println("查看账户方法...");
   } 
   @Override  
   public void update() {
       System.out.println("修改账户方法..."); 
   } 
}

/** 
 * 这是一个代理类（增强CountImpl实现类）   
 */ 
public class CountProxy implements Count {
  private CountImpl countImpl;   
  /**
   * 覆盖默认构造器   
   * @param countImpl
   */   
   public CountProxy(CountImpl countImpl) {
         this.countImpl = countImpl;    
   }   
   @Override 
   public void query() {
        System.out.println("处理之前");       
        countImpl.query(); // 调用委托类的方法;
        System.out.println("处理之后");     
   }  
   @Override  public void update() {
      System.out.println("处理之前");            
      countImpl.update();// 调用委托类的方法;         
      System.out.println("处理之后");  
   }
}
```

## 动态代理

### JDK动态代理

JDK1.3之后引入了动态代理机制，它是在程序运行时，运用反射机制动态创建代理类。
JDK动态代理的实现主要由一个类和一个接口组成，即java.lang.reflect.Proxy类和java.lang.reflect.InvocationHandler接口。

#### InvocationHandler接口

如下为InvocationHandler接口定义，InvocationHandler接口的实现类可以想象成一个代理的最终操作类。

```java
  public interface InvocationHandler {
        public Object invoke(Object proxy,Method method,Object[] args)throws Throwable;
  }
```

参数说明：

- Object proxy：指被代理的对象。
- Method method：要调用的方法
- Object[] args：方法调用时所需要的参数

#### Proxy类

Proxy类是专门完成代理的操作类，可以通过此类为一个或多个接口动态地生成实现类。
主要方法如下：

```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,InvocationHandler h)
          throws IllegalArgumentException     
```

参数说明：

- ClassLoader loader：类加载器
- Class<?>[] interfaces：得到全部的接口
- InvocationHandler h：得到InvocationHandler接口的子类实例

说明：在Proxy类中的newProxyInstance()方法中需要一个ClassLoader类的实例，ClassLoader实际上对应的是类加载器，在Java中主要有以下三种类加载器:

- Booststrap ClassLoader：此加载器采用C++编写，一般开发中是看不到的；
- Extendsion ClassLoader：用来进行扩展类的加载，一般对应的是jre\lib\ext目录中的类;
- AppClassLoader：(默认)加载classpath指定的类，是最常使用的是一种加载器

#### 代码示例

e.g.修改上述静态代理示例的代码

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

/** 
 * JDK动态代理代理类   
 */ 
public class CountInvocationHandler implements InvocationHandler {
   private Object target;//目标类
   public CountInvocationHandler  (Object target) {
         this.target = target;    
   }  

   @Override     
   /**     
    * 调用方法     
    */     
   public Object invoke(Object proxy, Method method, Object[] args)throws Throwable {    
        Object result=null;        
        System.out.println("开始....");                  
        result=method.invoke(target, args);//执行方法         
        System.out.println("结束...");         
        return result;     
   }  

  //测试
  public static void main(String[] args) {
        Count target = new CountImpl();
        CountInvocationHandler proxy = new CountInvocationHandler(target);
        //构造代理对象
        Count countProxy = (Count) Proxy.newProxyInstance(target.getClass().getClassLoader(),
				target.getClass().getInterfaces(), proxy);
        countProxy.update();
  }
}  
```
说明:
(1)通过Proxy为CountInvocationHandler生成相应的动态代理实例，当Proxy动态生成的代理对象上相应的接口方法被调用时，对应的 InvocationHandler就会拦截对应的方法调用，进行相应处理。
(2)InvocationHandler是实现横切逻辑的地方，即横切逻辑载体。
(3)若Count接口新增一个方法，静态代理的代理类也需要新增该方法的实现；而JDK动态代理则不需要。

#### 源码分析

Proxy.newProxyInstance(ClassLoader loader,Class<?>[] interfaces, InvocationHandler h)负责动态生成代理对象，这里就不贴源码了，自己去翻，只梳理一下重要步骤：

- 基础检查，例如h是否为null,是否有权限生成代理类；
- 通过getProxyClass0()【重要方法】查找或生成指定代理类的Class对象cl；若Proxy Class已经在缓存中则直接返回，否则通过ProxyClassFactory创建并放入缓存；
- 通过Class对象cl获取构造方法，利用反射通过Constructor.newInstance()方法,将h作为参数传入（即代理类的构造器以InvocationHandler作为参数），生成代理对象返回。

通过ProxyClassFactory创建Proxy Class通过该类的apply()方法，也不贴源码了，该方法的重要步骤如下：

- 遍历所有接口，进行一些检查，是否为接口，是否重复；
- 检查所有non-public interface是否在同一个包，不是则抛异常；
- 得到代理类名称，包名.$Proxy+递增不重复的数字，若含非public接口则用接口所在包为包名，否则用com.sun.proxy作为包名；
- 通过ProxyGenerator.generateProxyClass()方法生成字节码。

说明：可以通过如下代码设置ProxyGenerator生成的代理类的字节码保存，查看该代理类。

```java
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
```


<font color="red">由上知道，JDK动态代理机制只能对实现了相应Interface的类使用，若某个类没有实现任何的Interface,将无法使用动态代理机制为其生产相应的动态代理对象。</font>

### cglib动态代理

使用动态字节码生成技术扩展对象行为的原理是，对目标对象进行继承扩展，为其生成相应的子类，而子类可以通过override来扩展父类的行为；通过将横切逻辑的实现放到子类中，让系统使用扩展后的目标对象的子类，可以达到与代理模式相同的效果。
CGLIB动态字节码生成库可以在系统运行期间动态地为目标对象生成相应的扩展子类，且它可以对实现某种接口的类或没有实现任何接口的目标类进行扩展。
用CGLIB对目标类进行扩展，首先要实现net.sf.cglib.proxy.Callback；由于MethodIntercepteor扩展了Callback接口，所以通常直接使用MethodInterceptor接口。

e.g.cglib使用

```java
import java.lang.reflect.Method;   
import net.sf.cglib.proxy.Enhancer; 
import net.sf.cglib.proxy.MethodInterceptor; 
import net.sf.cglib.proxy.MethodProxy;   
/** 
* 使用cglib动态代理  
*/ 
public class CountCglib implements MethodInterceptor {   
   @Override     
   // 回调方法,封装横切逻辑    
   public Object intercept(Object obj,Method method, Object[] args,MethodProxy proxy)throws Throwable {
         System.out.println("开始....");
         proxy.invokeSuper(obj, args);         
         System.out.println("结束...");         
         return null;    
   }
   
   //测试
   public static void main(String[] args) {
       CountCglib cglib = new CountCglib ();
       // 创建代理对象
       Enhancer enhancer = new Enhancer();         
       enhancer.setSuperclass(CountImpl.class);//通过继承生成代理类
       enhancer.setCallback(cglib);// 回调方法
       CountImpl proxy = (CountImpl)enhancer.create();          
       proxy.update();     
  } 
} 
```
说明:通过CGLIB的Enhance为目标对象动态生成一个子类，并CglibCount实现的横切逻辑附加到子类中。
 注意:<font color="red">根据CGLIB的原理，可以知道CGLIB对类扩展的唯一限制是无法对final方法进行override。</font>

### Spring AOP的选择

默认情况下，若Spring AOP发现目标对象实现了相应Interface，则采用动态代理机制为其生产代理对象实例，若目标对象没有实现任何Interface或设置了proxy-target-class="true",Spring AOP会尝试使用一个称为CGLIB的开源的动态字节码生成类库，为目标对象生成动态代理对象实例。

可以查看Spring中DefaultAopProxyFactory.createAopProxy()方法说明了Spring AOP对两种动态代理的选择，源码如下：

```java
      public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		//ProxyConfig.isOptimize()=true 表示spring优化开启
		//ProxyConfig.isProxyTargetClass()=true表示配置了proxy-target-class="true"
		//hasNoUserSuppliedProxyInterfaces(ProxyConfig)=true表示bean没有实现接口或实现的接口是SpringProxy接口
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {//默认使用JDK动态代理
			return new JdkDynamicAopProxy(config);
		}
	}
```
