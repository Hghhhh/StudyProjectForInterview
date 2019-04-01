# ApplicationContext启动过程

以ClassPathXmlApplicationContext为例子，实例化ApplicationContext的时候我们调用了refresh这个方法。

```java
public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```

这个方法来自**org.springframework.context.support.AbstractApplicationContext.refresh()**

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
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

我们一步一步来refresh里面的过程

##1.prepareRefresh()

这个方法为ApplicationContext的refresh设置了开始时间、active状态，初始化context的环境资源，判断启动所需的属性是否都有了，还有就是对applicationListeners、earlyApplicationEvents的初始化。

简单来说，这个方法是为了启动ApplicationContext做准备的。

```java
/**
	 * Prepare this context for refreshing, setting its startup date and
	 * active flag as well as performing any initialization of property sources.
	 */
	protected void prepareRefresh() {
		// Switch to active.
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		if (logger.isDebugEnabled()) {
			if (logger.isTraceEnabled()) {
				logger.trace("Refreshing " + this);
			}
			else {
				logger.debug("Refreshing " + getDisplayName());
			}
		}

		// Initialize any placeholder property sources in the context environment.
		initPropertySources();

		// Validate that all properties marked as required are resolvable:
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// Store pre-refresh ApplicationListeners...
		if (this.earlyApplicationListeners == null) {
			this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
		}
		else {
			// Reset local application listeners to pre-refresh state.
			this.applicationListeners.clear();
			this.applicationListeners.addAll(this.earlyApplicationListeners);
		}

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```

## 2.ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory()（重要方法）

让子类刷新beanFactory，刷新的过程是关闭前一个bean工厂（如果有），以及为上下文生命周期的下一个阶段初始化一个新的bean工厂，刷新的过程进行了BeanDefintion的解析和注册，刷新完成之后返回新的beanFactory

	/**
	 * Tell the subclass to refresh the internal bean factory.
	 * @return the fresh BeanFactory instance
	 * @see #refreshBeanFactory()
	 * @see #getBeanFactory()
	 */
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		//让子类刷新beanFactory
		refreshBeanFactory();
		return getBeanFactory();
	}
	
		/**
		 * 实现过程是关闭前一个bean工厂（如果有），
		 以及为上下文生命周期的下一个阶段初始化一个新的bean工厂。
		 */
		@Override
		protected final void refreshBeanFactory() throws BeansException {
			if (hasBeanFactory()) {
			//销毁前面的Beans和BeanFactory
				destroyBeans();
				closeBeanFactory();
			}
			try {
				//创建一个新的BeanFactory
				DefaultListableBeanFactory beanFactory = createBeanFactory();
				//为新创建的BeanFactory设置初始信息
				beanFactory.setSerializationId(getId());
				//这个方法按照我们设置信息来配置BeanFactory，
				//包括allowBeanDefinitionOverriding、allowCircularReferences两个配置
				customizeBeanFactory(beanFactory);
				//这个方法进行了Resource资源定位、BeanDefinition的解析
				//还有将BeanDefinition注册到Ioc容器
				//会提供什么样的ResourceLoader由实现的子类决定
				loadBeanDefinitions(beanFactory);
				synchronized (this.beanFactoryMonitor) {
				//设置新的beanFactory
					this.beanFactory = beanFactory;
				}
			}
			catch (IOException ex) {
				throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
			}
		}
## 3.prepareBeanFactory(beanFactory)

如果我们只是简单的使用BeanFactory的话，其实单单做第二步：创建一个BeanFactory，用我们需要的ResourceLoader完成一些资源定位和BeanDefintion解析和注册就可以调用getBean来获取我们想要的Bean了（当然，如果只是使用BeanFactory的话ResourceLoader需要我们自己定义，而ApplicationContext为我们提供了特定的ResourceLoader），但是ApplicationContext在这个基础上还加了很多功能，从第三步开始就看出了。

简单来说，第三步是为前面创建的BeanFactory设置一些属性，比如classLoader、BeanPostProcessor等

```java
/**
	 *配置工厂的标准上下文特征，
	 *例如上下文的类加载器和后处理器。
	 * @param beanFactory the BeanFactory to configure
	 */
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// 为beanFactory设置当前ApplcationContext的classLoader还有一些各种设置。。
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
        //忽略自动连接的给定依赖接口。
        //这通常由应用程序上下文用于注册以其他方式解析的依赖项。
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

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

##4.postProcessBeanFactory(beanFactory)

这个方法在应用程序上下文的标准初始化之后为前面的BeanFactory注册BeanPostProcessor。这时候所有bean定义都将被加载，但还没有实例化bean。这允许在某些应用程序上下文中注册特殊的BeanPostProcessor。

这个方法是一个模板方法，具体由子类去实现它。

## 5.invokeBeanFactoryPostProcessors(beanFactory)

在实例化bean之前调用BeanFactoryPostProcessor的invokeBeanFactoryPostProcessors(beanFactory);对beanFactory进行一些处理。**这是一个很重要的扩展点，如果你想在Bean实例化前对BeanFactory进行处理的话，你就可以实现BeanFactoryPostProcessor接口。**

```java
	/**
	 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before singleton instantiation.
	 */
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```

## 6.registerBeanPostProcessors(beanFactory)

```java
	/**
	 * Instantiate and invoke all registered BeanPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before any instantiation of application beans.
	 */
	protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
	}
```

为BeanFacotory注册所有beanPostProcessor，如果你想在Bean初始化前后执行一些操作，你可以实现BeanPostProcessor接口，

注意：这个接口和上面的BeanFactoryPostProcessor的区别，BeanFactoryPostProcessor是在BeanDefinition构造完成之后，Bean还没开始实例化之前调用的，用来对BeanFactory进行一些设置，BeanPostProcessor是在Bean实例化的过程中调用的，对Bean进行一些操作或者设置。

```java
public interface BeanPostProcessor {
    	@Nullable
    //在bean实例化前执行操作
    //这里bean还没被实例化，是怎么把bean这个参数传进来的呢？
    //原理是使用了代理，创建了代理类来实现这个操作
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
    	@Nullable
    //在bean实例化后执行操作
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```

实现了BeanFactory，要规定BeanFactory的执行顺序，可以通过实现PriorityOrdered 或Ordered 来实现，执行顺序是**实现了priorityOrded>实现了orded>没有实现这个两个类中**(前面的BeanFactoryPostProcessor同理)

## 7.initMessageSource()

寻找beanName为messageSource的bean,并初始化之。如果没有，初始化默认的。用来做国际化功能;消息绑定,消息解析

## 8.initApplicationEventMulticaster()

寻找beanName为applicationEventMulticaster的bean，并初始化。如果没有，初始化默认的SimpleApplicationEventMulticaster。在广播时寻找所有实现接口ApplicationListener的类。

```java
	/**
	 * Initialize the ApplicationEventMulticaster.
	 * Uses SimpleApplicationEventMulticaster if none defined in the context.
	 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
	 */
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}
```

ApplicationEventMulticaster是事件广播器，把事件广播给ApplicationListener

```java
public interface ApplicationEventMulticaster {
	void addApplicationListener(ApplicationListener<?> listener);
	void addApplicationListenerBean(String listenerBeanName);
	void removeApplicationListener(ApplicationListener<?> listener);
	void removeApplicationListenerBean(String listenerBeanName);
	void removeAllListeners();
	void multicastEvent(ApplicationEvent event);
	void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);
}
```

## 9.onRefresh()

由子类来提供实现，在实例化单例之前调用特殊bean的初始化。

##10.registerListeners()

前面已经把事件广播器初始化好了，这下终于轮到**ApplicationListener**的注册了。**这是一个很重要的扩展点，如果你想对容器工作过程中发生的节点事件进行一些处理，比如容器要刷新、容器要关闭了，那么你就可以实现ApplicationListener**

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/applicationevent.png)

```java
	/**
	 * Add beans that implement ApplicationListener as listeners.
	 * Doesn't affect other listeners, which can be added without being beans.
	 */
	protected void registerListeners() {
		// Register statically specified listeners first.
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
        //寻找所有实现了ApplicationListener的bean
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
	//向广播器添加ApplicationListener
            getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
            //广播early application events
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}

//如果你想对容器工作过程中发生的节点事件进行一些处理，比如容器要刷新、容器要关闭了，那么你就可以实现ApplicationListener
@Component
public class MyApplicationListener implements ApplicationListener<ApplicationEvent> {

    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        System.out.println("-----收到应用事件：" + event);
    }
}
```

## 11.finishBeanFactoryInitialization(beanFactory)

实例化所有非懒加载的单例，如果我们没有给bean配置scope、lazy-init这些属性，默认就是非懒加载的单例bean。主要就是调用了getBean方法。调用链：getBean()->doGetBean()->createBean()->instantiate()->populateBean->applyPropertyValues()->resolveReference()，简单来说就是按照BeanDefinition的定义初始化bean和对bean进行依赖注入。

```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```

## 12.finishRefresh()

refreshd的最后一步，主要做了几件事：初始化LifecycleProcessor（生命周期管理器），调用生命周期管理器的onRefresh方法，发布ContextRefreshedEvent事件

```java
	/**
	 * Finish the refresh of this context, invoking the LifecycleProcessor's
	 * onRefresh() method and publishing the
	 * {@link org.springframework.context.event.ContextRefreshedEvent}.
	 */
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

## 13.resetCommonCaches()

重置Spring的公共反射元数据缓存,因为这部分数据已经不再需要了...

```java
	/**
	 * Reset Spring's common reflection metadata caches, in particular the
	 * {@link ReflectionUtils}, {@link AnnotationUtils}, {@link ResolvableType}
	 * and {@link CachedIntrospectionResults} caches.
	 * @since 4.2
	 * @see ReflectionUtils#clearCache()
	 * @see AnnotationUtils#clearCache()
	 * @see ResolvableType#clearCache()
	 * @see CachedIntrospectionResults#clearClassLoader(ClassLoader)
	 */
	protected void resetCommonCaches() {
		ReflectionUtils.clearCache();
		AnnotationUtils.clearCache();
		ResolvableType.clearCache();
		CachedIntrospectionResults.clearClassLoader(getClassLoader());
	}
```

##补充：

###LifecycleProcessor是干嘛的？

Lifecycle接口，任何spring管理的对象都可以实现这个接口，每个实现了这个接口的对象都有自己的生命周期需求，可以在spring容器启动和关闭的时候做一些操作，如下：

```
public interface Lifecycle {  
  void start();  
  void stop();  
  boolean isRunning();  
}  
```

那么，当ApplicationContext自身启动和停止时，它将自动调用上下文内所有生命周期的实现。通过委托给`LifecycleProcessor`来做这个工作。注意`LifecycleProcessor`自身扩展了Lifecycle接口。它也增加了两个其他的方法来与上下文交互，使得可以刷新和关闭。

```
public interface LifecycleProcessor extends Lifecycle {  
  void onRefresh();  
  void onClose();  
}  
```

利用这个Lifecycle可以实现一些扩展功能，但是本人几乎没有使用过，也是第一次听说bean可以实现Lifecycle的，Spring很确实强大。关于LifecycleProcessor和Lifecycle的具体实现可以看下面这篇文章中的例子：

[spring4.1.8扩展实战之四：感知spring容器变化(SmartLifecycle接口)](<https://blog.csdn.net/boling_cavalry/article/details/82051356>)



##启动的过程中抛出异常

到此为止，Spring的ApplcationContext就算启动起来了，如果启动过程有哪一步抛出了异常，catch异常的时候主要就是执行destroyBeans()销毁已经创建的singletons（具体就是执行ConcurrentHashMap.clear()，因为存储singtonbean的map是一个ConcurrentHashMap），然后还有Spring状态的改变。



## 用到的设计模式

模板方法设计模式、观察者模式、代理模式、策略模式、工厂模式（如果我们创建bena的时候指定了工厂的话）

关于Spring的设计模式，之后会再详细的总结一下。



最后总结一幅图：

(图片来自博客[框架源码系列八：Spring源码学习之Spring核心工作原理](https://www.cnblogs.com/leeSmall/p/10188408.html))

![refresh](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/spring/refresh%E8%BF%87%E7%A8%8B.png)