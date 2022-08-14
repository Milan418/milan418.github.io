---
title: IoC容器工作过程(下)
date: 2018-04-07 18:32:06
tags: Spring IoC
categories: Spring
keywords: Spring,Spring IoC,源码分析
description: 以XML配置方式为例，简单分析IoC容器工作过程，上一篇文章已经分析了IoC容器初始化的BeanFactory初始化，这一part讲Bean实例化。源码版本是Spring 4.3.x。
---

前面的 {% post_link IoC容器工作过程-上 %} 已经讲了BeanFactory的初始化和准备工作，终于到了refresh()方法中的最后步骤了-finishBeanFactoryInitialization方法，它实现了初始化所有的singleton beans（lazy-init除外）的过程。

```java
//AbstractApplicationContext.java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		//查看是否有conversionService这个示例，有则通过getBean(...)实例化并set
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
				@Override
				public String resolveStringValue(String strVal) {
					return getEnvironment().resolvePlaceholders(strVal);
				}
			});
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		//初始化 LoadTimeWeaverAware 类型的 Bean，
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		//初始化剩下的非懒加载的单例
		beanFactory.preInstantiateSingletons();
	}
```

这里我们就直接跳到对非lazy-init的单例的初始化-preInstantiateSingletons()方法。

```java
//DefaultListableBeanFactory.java
	public void preInstantiateSingletons() throws BeansException {
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {//遍历所有beanName
			//获得beanName对应的RootBeanDefinition
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {//非抽象、非懒加载的singleton加载
				//判断该beanName是否对应一个FactoryBean
				//FactoryBean-工厂类接口是spring提供的，用户可以通过实现该接口定制实例化 Bean 的逻辑
				//FactoryBean 适用于 Bean 的创建过程比较复杂的场景，比如数据库连接池的创建
				//Spring提供了70+个FactoryBean的实现
				if (isFactoryBean(beanName)) {
					//FactoryBean需要在 beanName 前面加上 ‘&’ 符号，调用getBean()实例化它
					final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							@Override
							public Boolean run() {
								return ((SmartFactoryBean<?>) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {//非FactoryBean，则直接调用getBean()实例化
					getBean(beanName);//具体实例化bean的逻辑
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		//非懒加载的singleton bean实例化完成，遍历判断是否为SmartInitializingSingleton
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged(new PrivilegedAction<Object>() {
						@Override
						public Object run() {
							smartSingleton.afterSingletonsInstantiated();
							return null;
						}
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

过程：

1. 获取BeanFactory中所有beanName;
2. 遍历所有beanName,获取beanName对应的BeanDefinition;
3. 若BeanDefinition对应一个非抽象、非懒加载的singleton则进行步骤4，否则直接跳过；
4. 判断BeanDefinition是否对应一个FactoryBean,否则进行步骤5；是则判断是否需要立马初始化，是则进行步骤5；
5. 通过getBean(String)方法具体实例化bean。

综上可知，<font color="red">容器初始化时只对非抽象、非懒加载的singleton进行初始化</font>，具体实例化bean的方法是getBean()方法,实现在AbstractBeanFactory中。

```java
//AbstractBeanFactory.java
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}

	protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {

		//beanName处理：1.FactoryBean带&;2.别名；
		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {//检查bean是否已实例化&args==null(表示只是需要获取bean)
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			//若是普通bean则直接返回，否则返回FactoryBean创建的实例
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) { 
				//当前线程正在创建此 beanName 的 prototype 类型的 bean则异常
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			//不能查找到BeanDefinition,则尝试用父级BeanFactory
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}
			
			//以下创建bean的逻辑
			if (!typeCheckOnly) {
				//将bean放入一个 alreadyCreated 的 Set 集合中
				markBeanAsCreated(beanName);
			}
			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				//检查depends-on中依赖，先初始化需要依赖的bean（销毁则相反）
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						getBean(dep);
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {//singleton创建
					//说明该getSingleton方法会先去ConcurrentHashMap中查找该beanName对应的bean,有则返回；
					//没有则通过工厂类ObjectFactory.getObject()创建bean，并放到Map中
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							try {
								return createBean(beanName, mbd, args);//创建bean
							}
							catch (BeansException ex) {
								// Explicitly remove instance from singleton cache: It might have been put there
								// eagerly by the creation process, to allow for circular reference resolution.
								// Also remove any beans that received a temporary reference to the bean.
								destroySingleton(beanName);
								throw ex;
							}
						}
					});
					//若是普通bean则直接返回，否则返回FactoryBean创建的实例
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {//prototype实例创建
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						//前置操作，将beanName放到ThreadLocal中
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);//创建bean
					}
					finally {
						//后置操作，将beanName移出ThreadLocal
						afterPrototypeCreation(beanName);
					}
					//若是普通bean则直接返回，否则返回FactoryBean创建的实例
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {//非singleton或prototype则委托给对应的scope来创建
					//先获取对应的Scope对象
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						//从scope对象中查找beanName对应的bean
						//例如：ServletContext scope 存储在servletContext 的 attribute属性中
						Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
							@Override
							public Object getObject() throws BeansException {
								beforePrototypeCreation(beanName);
								try {
									return createBean(beanName, mbd, args);//创建bean
								}
								finally {
									afterPrototypeCreation(beanName);
								}
							}
						});
                        //若是普通bean则直接返回，否则返回FactoryBean创建的实例
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		//类型检查
		if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

过程：

1. 对beanName进行处理，考虑FactoryBean（beanName以&开头）和别名；
2. 通过beanName获取实例，检查是否已经创建过，是则返回该实例或FactoryBean创建的实例对象；否则进入步骤3；
3. 判断当前线程是否创建过beanName对应的prototype bean,是则异常；否则进入步骤4；
4. 获取父级BeanFactory,若有父级容器，则依赖父级容器完成getBean()操作；没有则进入步骤5；
5. 注册depends-on定义的依赖关系，并初始化依赖；
6. 根据scope类型初始化bean。对于singleton和propotype的bean都是通过createBean方法创建bean,而其它scope则委托给对应的Scope类处理。

## bean实例化
接下来就到singleton和propotype bean实例化的createBean方法了，该方法由抽象类AbstractBeanFactoryde子类AbstractAutowireCapableBeanFactory实现。

```java
//AbstractAutowireCapableBeanFactory.java
	protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		//确保bean对应的Class被加载
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		//方法覆盖，对<bean>中定义的<lookup-method /> 和 <replaced-method />的具体处理
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			//让 BeanPostProcessor 在这一步有机会返回代理，而不是 bean 实例
			//具体查看 InstantiationAwareBeanPostProcessor 接口
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}
		//【重要方法】
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
```

继续往下，到创建bean的doCreateBean方法。

```java
//AbstractAutowireCapableBeanFactory.java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {//非FactoryBean实例，则创建实例
			instanceWrapper = createBeanInstance(beanName, mbd, args);//重要方法-创建bean
		}
		//bean实例
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
		//bean类型
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
		mbd.resolvedTargetType = beanType;

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		//循环依赖解决
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			//bean属性赋值
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
				//处理 bean 初始化完成后的各种回调，例如init-method方法, InitializingBean接口和BeanPostProcessor 接口
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```
过程：
1. createBeanInstance方法创建指定类的实例；
2. populateBean方法进行依赖注入；
3. initializeBean方法回调init-method指定方法，InitializingBean接口，BeanPostProcessor接口。

### 创建Bean实例

先看createBeanInstance方法。

```java
//AbstractAutowireCapableBeanFactory.java
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
		// Make sure bean class is actually resolved at this point.
		//确保已加载bean的Class
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		//Class未加载或类没有访问权限
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		if (mbd.getFactoryMethodName() != null)  {//工厂方法实例化
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		//判断bean是否为re-createing
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {//是re-creating bean
			if (autowireNecessary) {
				//构造函数实例化
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				//默认无参构造函数实例化
				return instantiateBean(beanName, mbd);
			}
		}

		//非re-creating bean
		// Need to determine the constructor...
		//确认实例化bean是否采用有参函数构造函数构建-有有参构造函数或autowire=constructor或bean设置了构造函数参数值
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		//否则采用无参构造函数实例化
		// No special handling: simply use no-arg constructor.
		return instantiateBean(beanName, mbd);
	}
```
说明：
1. instantiateUsingFactoryMethod方法通过工厂方法实例化bean，有两种形式：
    * 工厂类（如下），则这种情况下需要先实例化该工厂类；
      ```xml
      <!-- 由Spring容器中已经存在的bean的非static工厂方法来初始化一个new bean-->
      <!--class属性为空，factory-bean属性值为包含普通工厂方法的已定义的bean-->
	  <!--factory-method属性则指定普通工厂方法 -->
      <bean id="serviceLocator" class="com.milan.DefaultServiceLocator" />
      <bean id="serviceClient" factory-bean="serviceLocator" factory-method="createInstance">
      ```
    * 静态工厂方法（如下）
      ```xml
      <!-- class指定静态工厂方法所在类，factory-method指定该静态工厂方法-->
      <bean id="clientService" class="com.milan.ClientService" factory-method="createInstance">
      ```
2. autowireConstructor和instantiateBean方法分别表示有参构造函数实例化和默认构造函数实例化过程。
内部都是通过反射实现初始化。

### bean属性注入
这里就只贴一下代码。
```java
	protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
		PropertyValues pvs = mbd.getPropertyValues();

		if (bw == null) {//null实例则跳过或抛异常
			if (!pvs.isEmpty()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				// Skip property population phase for null instance.
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
		boolean continueWithPropertyPopulation = true;

		//若实现了InstantiationAwareBeanPostProcessor，则在这里对 bean进行状态修改
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						continueWithPropertyPopulation = false;
						break;
					}
				}
			}
		}

		if (!continueWithPropertyPopulation) {
			return;
		}

		//bean的autowire=byName 或 autowire=byType,则分别根据autowire注入依赖
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
				mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

			// Add property values based on autowire by name if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}

			// Add property values based on autowire by type if applicable.
			if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}

			pvs = newPvs;
		}

		//依赖检查
		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

		if (hasInstAwareBpps || needsDepCheck) {
			PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			if (hasInstAwareBpps) {
				for (BeanPostProcessor bp : getBeanPostProcessors()) {
					if (bp instanceof InstantiationAwareBeanPostProcessor) {
						InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
						pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvs == null) {
							return;
						}
					}
				}
			}
			if (needsDepCheck) {
				checkDependencies(beanName, mbd, filteredPds, pvs);
			}
		}

		//将属性值赋值到bean
		applyPropertyValues(beanName, mbd, bw, pvs);
	}
```

### 初始化回调
这个过程比较容易理解，直接贴代码吧。
```java
	protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				@Override
				public Object run() {
					invokeAwareMethods(beanName, bean);
					return null;
				}
			}, getAccessControlContext());
		}
		else {
			 //若bean 实现了 BeanNameAware、BeanClassLoaderAware 或 BeanFactoryAware 接口，回调
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			//BeanPostProcessor 的 postProcessBeforeInitialization 回调
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			//首先若 bean 实现了 InitializingBean 接口，调用 afterPropertiesSet() 方法
			//再者若 bean 中定义的 init-method，则调用该初始化方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}

		if (mbd == null || !mbd.isSynthetic()) {
			// BeanPostProcessor 的 postProcessAfterInitialization 回调
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
		return wrappedBean;
	}
```

## 总结
本文参考了[Spring IOC 容器源码分析](http://www.importnew.com/27469.html)。
