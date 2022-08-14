---
layout: w
title: IoC容器工作过程(上)
date: 2018-04-07 09:14:34
tags: Spring IoC
categories: Spring
keywords: Spring,Spring IoC,源码分析
description: 以XML配置方式为例，简单分析IoC容器工作过程，包括IoC容器启动，资源文件读取，Bean定义解析为BeanDefinition并注册，bean实例创建，bean获取等。本文先分析第一part:BeanFactory实例化和其它准备工作，下篇再讲Bean实例创建等。源码版本是Spring 4.3.x。
---

以XML配置文件方式为例，容器启动代码如下。

```java
public static void main(String[] args) {
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");
}
```

所以先看ClassPathXmlApplicationContext构造方法。

```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
  //将配置文件抽象为Resource,存放到该数组中
  private Resource[] configResources;
 
  //已经有ApplicationContext并需要配置成父子关系则使用该构造方法
  public ClassPathXmlApplicationContext(ApplicationContext parent) {
    super(parent);
  }
  ...
  public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
      throws BeansException {
    super(parent);

    setConfigLocations(configLocations);
    if (refresh) {
      refresh(); // 核心方法
    }
  }
  ...
}
```

<font color="red">refresh()方法即IoC容器构建的入口，该方法实现在父类AbstractApplicationContext中。</font>

```java
//AbstractApplicationContext.java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		//获取锁才能执行Application启动或重建操作
		synchronized (this.startupShutdownMonitor) {
			// 准备工作：标记变量、启动时间重置；处理配置文件...
			prepareRefresh();

			//获取一个beanFactory实例,解析bean定义成BeanDefinition,并注册到BeanDefinitionRegistry中
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			//设置BeanFactory的一些属性，例如classLoader,post-processors等
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				//这里所有的 Bean 都加载、注册完成了，但是都还没有初始化
				//添加一些特殊的 BeanPostProcessor的实现类
				postProcessBeanFactory(beanFactory);

				//调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory)
				invokeBeanFactoryPostProcessors(beanFactory);

				//注册 BeanPostProcessor 的实现类,此时bean并未初始化
				//此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
				// 两个方法分别在 Bean 初始化之前和初始化之后得到执行。
				registerBeanPostProcessors(beanFactory);

				// 初始化当前 ApplicationContext 的 MessageSource--国际化
				initMessageSource();

				//初始化当前 ApplicationContext 的事件广播器
				initApplicationEventMulticaster();

				//由具体的子类实现，用来初始化一些特殊的bean(在初始化singleton bean之前)
				onRefresh();

				// 注册事件监听器，监听器需要实现 ApplicationListener 接口
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				//初始化所有的singleton beans,lazy-init除外
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				//广播事件
				finishRefresh();
			}
			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				//销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```
以下对refresh()中的重要步骤进行一一说明。

# 初始化IoC容器准备

该步骤主要是对IoC容器的一些标识变量进行初始化。查看AbstractApplicationContext类中的prepareRefresh()方法，如下：

```java
//AbstractApplicationContext.java
	protected void prepareRefresh() {
		//标记变量closed，active（均为AtomicBoolean类型）、启动时间重置
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isInfoEnabled()) {
			logger.info("Refreshing " + this);
		}

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
		//校验XML文件
		getEnvironment().validateRequiredProperties();

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<ApplicationEvent>();
	}
```

在refresh()方法中注意到只有获取锁，才能执行容器重建操作。

# 获取BeanFactory实例

上述准备工作完成后,则获取BeanFactory实例，该过程包含如下几步：

1. 判断是否持有BeanFactory,有则销毁（销毁所有beans,beanFactory实例置为null）;
2. 创建一个BeanFactory实例；
3. 设置BeanFactory属性--allowBeanDefinitionOverriding(是否允许Bean覆盖)和allowCircularReferences（是否允许循环引用）；
4. 将Bean定义解析为BeanDefinition，并将BeanDefinition注册到BeanDefinitionRegistry（一个HashMap）上。
5. 返回该BeanFactory.

先查看AbstractApplicationContext的obtainFreshBeanFactory()方法。

```java
//AbstractApplicationContext.java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		//由子类例如AbstractRefreshableApplicationContext负责实现该方法
		//该方法主要实现了BeanFactory重建，解析Bean定义成BeanDefinition，并注册到BeanDefinitionRegistry
		refreshBeanFactory();
		//返回刚刚创建的 BeanFactory
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```

这里直接跳到重要的方法refreshBeanFactory()。

```java
//AbstractRefreshableApplicationContext.java
	protected final void refreshBeanFactory() throws BeansException {
		//1.判断是否持有BeanFactory,有则销毁
		if (hasBeanFactory()) {
			destroyBeans();//销毁所有beans
			closeBeanFactory();//将beanFactory置为null
		}
		try {
			//2.创建一个新的BeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());//序列化BeanFactory时需要
			//3.设置BeanFactory属性-allowBeanDefinitionOverriding(是否允许Bean覆盖)和allowCircularReferences（是否允许循环引用）
			customizeBeanFactory(beanFactory);
			//4.解析bean配置信息-->BeanDefinition，并将beanName-BeanDefinition注册到BeanDefinitionRegistry(一个HashMap)
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
```

## BeanFactory实例初始化

前面已经讲了重要步骤了，这里就不重复了，只是说明几个点。

* ApplicationContext内部持有一个BeanFactory实例，该实例是DefaultListableBeanFactory，它是BeanFactory的最终默认实现类，实现了BeanFactory的三个子类。
* 默认情况下，allowBeanDefinitionOverriding 属性为 null,所以在同一配置文件中定义 bean 时使用了相同的 id 或 name会抛错，不同配置文件则会发生覆盖。
* Spring允许循环依赖，例如A 依赖 B，而 B 依赖 A；但A构造方法中依赖B,B构造方法中依赖A，这种是不行的

## 加载&注册Bean

前面BeanFactory实例初始化后，到loadBeanDefinitions(DefaultListableBeanFactory)该重要方法，该方法完成Bean解析为BeanDefinition和BeanDefinition注册到BeanFactory中,方法如下：

```java
//AbstractXmlApplicationContext.java
	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		//为给定BeanFactory实例化一个XmlBeanDefinitionReader
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		//初始化BeanDefinitionReader,由具体子类实现
		initBeanDefinitionReader(beanDefinitionReader);
		//解析XML文件 为BeanDefinition,并注册
		loadBeanDefinitions(beanDefinitionReader);
	}
```

在正式分析上面过程之前先介绍BeanDefintion。

### BeanDefinition定义

Bean配置信息会解析成BeanDefinition,BeanDefinition 中保存了我们的 Bean 信息，比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖了哪些 Bean。Spring中BeanDefinition定义如下

```java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
	//作用域
	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

	int ROLE_APPLICATION = 0;
	int ROLE_SUPPORT = 1;
	int ROLE_INFRASTRUCTURE = 2;

	//父Bean
	void setParentName(String parentName);
	String getParentName();

	//bean的类名
	void setBeanClassName(String beanClassName);
	String getBeanClassName();

	//bean scope
	void setScope(String scope);
	String getScope();

	//是否懒加载
	void setLazyInit(boolean lazyInit);
	boolean isLazyInit();
	
	//设置该 Bean 依赖的所有的 Bean，是 depends-on="" 属性设置的值
	void setDependsOn(String... dependsOn);
	String[] getDependsOn();

	// 设置该 Bean 是否可以注入到其他 Bean 中，只对根据类型注入有效，
	// 如果根据名称注入，即使这边设置了 false，也是可以的
	void setAutowireCandidate(boolean autowireCandidate);
	boolean isAutowireCandidate();

	//同一接口的多个实现，若不指定名字，Spring 优先选择设置primary = true 的
	void setPrimary(boolean primary);
	boolean isPrimary();

	//Bean采用工厂方法实例化，指定工厂类或工厂方法名
	void setFactoryBeanName(String factoryBeanName);
	String getFactoryBeanName();

	void setFactoryMethodName(String factoryMethodName);
	String getFactoryMethodName();

	// 构造器参数
	ConstructorArgumentValues getConstructorArgumentValues();
	MutablePropertyValues getPropertyValues();

	// Read-only attributes
	//作用域
	boolean isSingleton();
	boolean isPrototype();
	//是否为抽象类
	boolean isAbstract();

	int getRole();
	String getDescription();
	String getResourceDescription();
	BeanDefinition getOriginatingBeanDefinition();
}
```

### Bean解析

继续AbstractXmlApplicationContext的loadBeanDefinitions(DefaultListableBeanFactory)，该方法中先初始化一个 XmlBeanDefinitionReader，然后通过loadBeanDefinitions(XmlBeanDefinitionReader)方法来解析XML文件。如下为loadBeanDefinitions(XmlBeanDefinitionReader)方法：

```java
//AbstractXmlApplicationContext.java
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
```

由上可以看出Spring将配置资源文件获取后抽象为Resource,XmlBeanDefinitionReader遍历这些Resource,然后调用loadBeanDefinitions(Resource)依次解析，解析过程是先将该Resource encode并放到ThreadLocal中，然后调用doLoadBeanDefinitions(InputStream, Resource)解析Bean定义。这里就直接跳到核心方法doLoadBeanDefinitions(InputStream, Resource)。

```java
//XmlBeanDefinitionReader.java
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			//解析DOM文件为Document
			Document doc = doLoadDocument(inputSource, resource);
			//将Document中包含BeanDefinition注册
			return registerBeanDefinitions(doc, resource);
		}
		catch (...
	}

	//重要方法
	public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		//实例化一个Reader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		//获取已有的BeanDefinition的数目
		int countBefore = getRegistry().getBeanDefinitionCount();
		//从document中去读取BeanDefinition,并注册到ReaderContext
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		//返回本次解析的BeanDefinition的数目
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

由上可知具体document中读取BeanDefinition由BeanDefinitionDocumentReader的registerBeanDefinitions方法实现，BeanDefinitionDocumentReader是一个接口，默认实现类是DefaultBeanDefinitionDocumentReader，所以直接定位到DefaultBeanDefinitionDocumentReader的registerBeanDefinitions方法，该方法实现从根节点开始遍历各个节点，解析节点为BeanDefinition并注册。

```java
//DefaultBeanDefinitionDocumentReader.java
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		Element root = doc.getDocumentElement();//获取根节点
		//从根节点开始遍历，解析Bean定义为BeanDefinition，并注册
		doRegisterBeanDefinitions(root);
	}

	protected void doRegisterBeanDefinitions(Element root) {
		//解析Bean定义的重要类
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {//namespace是否空或http://www.springframework.org/schema/beans
			//查看根节点 <beans ... profile="dev" /> 中的 profile 是否是当前环境需要的，不需要的则不作处理直接返回
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);//钩子方法，子类实现
		parseBeanDefinitions(root, this.delegate);//解析
		postProcessXml(root);//钩子方法，子类实现

		this.delegate = parent;
	}
```

<strong>说明：</strong>
profile指不同的环境配置，我们可以在&lt;beans profile="dev,prod">定义环境配置，Spring 在启动的过程中，会去寻找 “spring.profiles.active” 的属性值，该属性值可以通过操作系统环境变量、JVM 系统变量、web.xml 中定义的参数、JNDI等配置。
Spring boot中通过application.properties 配置各个环境通用的配置，application-{profile}.properties 中配置特定环境的配置，然后在启动的时候指定 profile。
@ActiveProfiles也可以在测试时指定应用的profile。

继续回到Bean解析，核心解析方法是parseBeanDefinitions方法，该方法针对default namespace中的标签（包括&lt;import>,&lt;alias>,&lt;bean>,&lt;beans>）和其它namespace下的标签采用不同的解析方式。

```java
//DefaultBeanDefinitionDocumentReader.java
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {//namespace是否空或http://www.springframework.org/schema/beans
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {//遍历子节点
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						//解析默认schema定义的<import />、<alias />、<bean />、<beans />节点
						parseDefaultElement(ele, delegate);
					}
					else {
						//解析 <mvc />、<task />、<context />、<aop />
						//其内部是通过相应的parser来解析
						//MvcNamespaceHandler、TaskNamespaceHandler、ContextNamespaceHandler、AopNamespaceHandler
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {//解析 <mvc />、<task />、<context />、<aop />
			delegate.parseCustomElement(root);
		}
	}
```

这里我们就只看bean的解析，即parseDefaultElement方法,如下：

```java
//BeanDefinitionParserDelegate.java
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {//处理<import>标签，load该标签引入资源文件中的BeanDefinition
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {//处理<alias>标签,将name-alias注册
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {//处理<bean>标签
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {//处理<beans>标签，递归
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```

这里我们也只看重要的processBeanDefinition方法，它负责解析&lt;bean>标签为BeanDefinition并注册。

```java
//BeanDefinitionParserDelegate.java
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		//解析Element为BeanDefinition
		//BeanDefinitionHolder包含BeanDefinition,beanName和alias数组
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				//将beanName-BeanDefinition键值对到BeanDefinitionRegistry
				//由BeanDefinitionRegistry的实现类-例如DefaultListableBeanFactory等具体实现
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			//发送一个registration event
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

BeanDefinitionParserDelegate的parseBeanDefinitionElement(Element)是最终解析Bean定义的方法,该方法返回一个BeanDefinitionHolder，它包含一个 BeanDefinition 的实例和它的 beanName、aliases数组这三个信息。

```java
//BeanDefinitionParserDelegate.java
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}

    public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
		//获取<bean>的id,name属性
		String id = ele.getAttribute(ID_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

		//name设置多个值时拆分，分隔号可以是空格、逗号，分号
		List<String> aliases = new ArrayList<String>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

		String beanName = id;//beanName默认是id
		//若定义了name属性，则beanName为第一个name值，其余name值为别名
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isDebugEnabled()) {
				logger.debug("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
            //判断beanName是否已被其它bean用作名称或别名
			checkNameUniqueness(beanName, aliases, ele);
		}
		
		//解析为Element为BeanDefinition
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {//若未设置id或name,则生成beanName
				try {
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}
```
过程分析：
1. 获取&lt;bean>的id,name属性,id若存在则默认为beanName,否则若定义了name属性，则beanName为第一个name值，其余name值为别名；
2. 将bean定义解析为BeanDefinition；
3. 若没有beanName(即未设置id和name)，则生成beanName;
4. 将BeanDefinition,beanName,alias封装成BeanDefinitionHolder返回。

真正将bean定义解析为BeanDefinition要看parseBeanDefinitionElement(Element,String,BeanDefinition)方法，如下：

```java
//BeanDefinitionParserDelegate.java
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}

		try {
			String parent = null;
			if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
				parent = ele.getAttribute(PARENT_ATTRIBUTE);
			}
			//根据className,Parent实例化BeanDefinition
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);
			
			//解析ele的属性 对应到给定 bd
            //例如lazy-init属性，autowire属性,depends-on属性，scope属性等
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

			// 解析 <meta />
			parseMetaElements(ele, bd);
			//解析 <lookup-method />
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			//解析 <replaced-method />
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

			// 解析 <constructor-arg />
			parseConstructorArgElements(ele, bd);
			parsePropertyElements(ele, bd);//解析 <property />
			parseQualifierElements(ele, bd); // 解析 <qualifier />

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```
这里我们看一下对bean其它属性的解析-parseBeanDefinitionAttributes方法。

```java
	public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
			BeanDefinition containingBean, AbstractBeanDefinition bd) {

		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {//老的singleton属性
			error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
		}
		else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
			bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
		}
		else if (containingBean != null) {
			// Take default from containing bean in case of an inner bean definition.
			bd.setScope(containingBean.getScope());
		}

		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
		}

		//懒加载属性，default或true均表示需要懒加载
		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
		if (DEFAULT_VALUE.equals(lazyInit)) {//若值为default，则沿用整个<beans>的设置
			lazyInit = this.defaults.getLazyInit();
		}
		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

		//autowire属性值有default(默认值),byName,byType,constructor,autodetect
		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
		bd.setAutowireMode(getAutowireMode(autowire));
		
		//dependency-check属性，取值有all，objects，simple；不设置则表示不检查
		String dependencyCheck = ele.getAttribute(DEPENDENCY_CHECK_ATTRIBUTE);
		bd.setDependencyCheck(getDependencyCheck(dependencyCheck));

		//depends-on属性
		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
		}

		//autowire-candidate属性，默认true；
		//该属性有设置则取设置值，若该属性未设置或default，则考虑取<beans>设置的值，若<beans>未设置则默认true
		String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
		if ("".equals(autowireCandidate) || DEFAULT_VALUE.equals(autowireCandidate)) {
			String candidatePattern = this.defaults.getAutowireCandidates();
			if (candidatePattern != null) {
				String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
				bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
			}
		}
		else {
			bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
		}
		
		//primary属性
		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
		}

		//init-method属性，未设置时则取<beans>上的设置
		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
			if (!"".equals(initMethodName)) {
				bd.setInitMethodName(initMethodName);
			}
		}
		else {
			if (this.defaults.getInitMethod() != null) {
				bd.setInitMethodName(this.defaults.getInitMethod());
				bd.setEnforceInitMethod(false);
			}
		}

		//init-method属性，未设置时则取<beans>上的设置
		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
			bd.setDestroyMethodName(destroyMethodName);
		}
		else {
			if (this.defaults.getDestroyMethod() != null) {
				bd.setDestroyMethodName(this.defaults.getDestroyMethod());
				bd.setEnforceDestroyMethod(false);
			}
		}
		
		//factory-method属性
		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
		}
		//factory-bean属性
		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
		}

		return bd;
	}
```
由上面的解析过程可以知道，<font color="red">懒加载属性&lt;bean>优先级最高，再是&lt;beans>，而默认值是false</font>。

### BeanDefinition注册

通过BeanDefinitionReaderUtils.registerBeanDefinition(BeanDefinitionHolder， BeanDefinitionRegistry)方法将BeanDefiniton注册到BeanDefinitionRegistry,若有别名，则将alias -> beanName存储到一个map中，通过别名获取该bean时，则先获取该别名对应的beanName。

BeanDefinitionRegistry是一个接口，DefaultListableBeanFactory和GenericApplicationContext均是其实现类。如下以DefaultListableBeanFactoryz为例，说明其具体实现。

```java
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition oldBeanDefinition;

		oldBeanDefinition = this.beanDefinitionMap.get(beanName);
		if (oldBeanDefinition != null) {//该bean已注册过
			if (!isAllowBeanDefinitionOverriding()) {//若不允许覆盖则抛异常
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + oldBeanDefinition + "] bound.");
			}
			else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (this.logger.isWarnEnabled()) {
					this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							oldBeanDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(oldBeanDefinition)) {//否则覆盖已注册的bean
				if (this.logger.isInfoEnabled()) {
					this.logger.info("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + oldBeanDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			//覆盖操作
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {//该bean未注册过
			if (hasBeanCreationStarted()) {//已经有 Bean 开始初始化了，则使用同步锁添加
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					//将该beanName-beanDefinition放到map,并更新beanDefinitionNames
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					//在 Spring 容器启动的最后，会预初始化 所有的 singleton beans，manualSingletonNames保存singleton的注册顺序
					//若manualSingletonNames包含beanName，则需要删除
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {//还在注册阶段，则直接更新
				// Still in startup registration phase
				// 将 BeanDefinition 放到beanDefinitionMap这个 map 中，beanDefinitionMap保存了所有的 BeanDefinition
				this.beanDefinitionMap.put(beanName, beanDefinition);
				//beanDefinitionNames是个 ArrayList，所以会按照 bean 配置的顺序保存每一个注册的 Bean 的名字
				this.beanDefinitionNames.add(beanName);
				//manualSingletonNames是一个LinkedHashSet，代表的是手动注册的 singleton bean
				//这里的bean都不是手动注册的，所以要remove
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		//若该bean重新覆盖或注册成功，则reset
		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
```

说明：
1. 若允许覆盖，则新的BeanDefinition直接覆盖旧的；
2. beanName-->BeanDefinition其实是注册到一个Map中，该Map保存了所有的BeanDefinition。

# 其它准备

经过如上过程，BeanFactory实例化，bean定义解析为BeanDefinition,且beanName-BeanDefinition注册到BeanDefinitionRegistry，即所有Bean都加载、注册完成并未实例化。
接下来还有一系列对BeanFactory的操作,这里先再粘贴一遍refresh()方法。

```java
//AbstractApplicationContext.java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		//获取锁才能执行Application启动或重建操作
		synchronized (this.startupShutdownMonitor) {
			// 准备工作：标记变量、启动时间重置；处理配置文件...
			prepareRefresh();

			//获取一个beanFactory实例,解析bean定义成BeanDefinition,并注册到BeanDefinitionRegistry中
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			//设置BeanFactory的一些属性，例如classLoader,post-processors等
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				//这里所有的 Bean 都加载、注册完成了，但是都还没有初始化
				//添加一些特殊的 BeanPostProcessor的实现类
				postProcessBeanFactory(beanFactory);

				//调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory)
				invokeBeanFactoryPostProcessors(beanFactory);

				//注册 BeanPostProcessor 的实现类,此时bean并未初始化
				//此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
				// 两个方法分别在 Bean 初始化之前和初始化之后得到执行。
				registerBeanPostProcessors(beanFactory);

				// 初始化当前 ApplicationContext 的 MessageSource--国际化
				initMessageSource();

				//初始化当前 ApplicationContext 的事件广播器
				initApplicationEventMulticaster();

				//由具体的子类实现，用来初始化一些特殊的bean(在初始化singleton bean之前)
				onRefresh();

				// 注册事件监听器，监听器需要实现 ApplicationListener 接口
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				//初始化所有的singleton beans,lazy-init除外
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				//广播事件
				finishRefresh();
			}
			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				//销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```
## prepareBeanFactory
 
该方法用来设置BeanFactory的classLoader属性,添加几个post-processors等，手动注册Environment实例：

```java
//AbstractApplicationContext.java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		//设置BeanFactory的类加载器
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		//添加一个BeanPostProcessor
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		//自动装配时忽略依赖以下接口
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		//为以下几种类型的 bean注入对应的值
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		//添加一个BeanPostProcessor，在 bean 实例化后，如果是 ApplicationListener 的子类，
		// 那么将其添加到 listener 列表中，即注册事件监听器
		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		//注册Environment示例，若没有定义，则默认注册一个
		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

## postProcessBeanFactory

该方法用来添加一些特殊的 BeanPostProcessor的实现类,该方法由具体的ApplicationContext实现类实现，例如AbstractRefreshableWebApplicationContext中，该方法用来添加ServletContextAwareProcessor实例。

## invokeBeanFactoryPostProcessors

按照给定顺序实例化所有已注册的BeanFactoryPostProcessor bean，并调用其postProcessBeanFactory方法。

```java
//AbstractApplicationContext.java
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		//实例化并调用BeanFactoryPostProcessor的postProcessBeanFactory(BeanFactory)
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```
<font color="red">Spring IoC容器允许BeanFactoryPostProcessor在容器实例化任何bean之前读取bean的定义(配置元数据)，并可以修改它</font>。
同时可以定义多个BeanFactoryPostProcessor，通过设置'order'属性来确定各个BeanFactoryPostProcessor执行顺序。这里查看PostProcessorRegistrationDelegate的invokeBeanFactoryPostProcessors方法中查看对Order的处理。

```java
//PostProcessorRegistrationDelegate.java
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
	    ...
	    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
}
```
从源码看出 实现了PriorityOrdered接口的优先级最高，其次是实现Ordered接口的。

## registerBeanPostProcessors

按照给定顺序实例化所有已注册的BeanPostProcessor bean，也是由PostProcessorRegistrationDelegate工具类来实现的，类似前面的BeanFactoryPostProcessor处理。

```java
//PostProcessorRegistrationDelegate.java
	public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```

## 其它
其它还有一些国际化，事件广播器注册就暂时不详细说明。
