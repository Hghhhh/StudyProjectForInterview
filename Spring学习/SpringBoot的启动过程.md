##前言：

SpringBoot的特点：
官方文档对它的描述是BUILD ANYTHING，可以简化配置，提供开发效率

- 独立运行Spring项目
  Spring boot 可以以jar包形式独立运行，运行一个Spring Boot项目只需要通过java -jar xx.jar来运行。

- 内嵌servlet容器
  Spring Boot可以选择内嵌Tomcat、jetty或者Undertow,这样我们无须以war包形式部署项目。

- 提供starter简化Maven配置
  spring提供了一系列的start pom来简化Maven的依赖加载。

- 自动装配Spring 
  SpringBoot会根据在类路径中的jar包、类，为jar包里面的类自动配置Bean，这样会极大地减少我们要使用的配置。当然，SpringBoot只考虑大多数的开发场景，并不是所有的场景，若在实际开发中我们需要配置Bean，而SpringBoot灭有提供支持，则可以自定义自动配置。例如说我们在类路径中加入了Spring-Security的jar包，SpringBoot会自动为我们创建一个java base config，无需我们自己创建。 

- 准生产的应用监控
  SpringBoot提供基于http ssh telnet对运行时的项目进行监控。

- 无xml配置　　

  SpringBoot不是借助xml配置生成来实现的，而是通过java配置和注解配置结合来实现的，这是Spring4.x提供的新特性。





# SpringBoot的启动

启动SpringBoot我们一般会使用下面的代码：

```java
@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```

##@SpringBootApplication

```java
@Target(ElementType.TYPE) //注解的适用范围，其中TYPE用于描述类、接口（包括包注解类型）或enum声明
@Retention(RetentionPolicy.RUNTIME) //注解的生命周期，保留到class文件中
@Documented //表明这个注解应该被javadoc记录
@Inherited // 子类可以继承该注解
@SpringBootConfiguration // 继承了Configuration，表示当前是注解类
@EnableAutoConfiguration // 开启springboot的自动注解功能，借助@import的帮助
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })  // 扫描路径设置
public @interface SpringBootApplication {
    ...
}
```

虽然定义使用了多个Annotation进行了原信息标注，但实际上重要的只有三个Annotation：

- @Configuration（@SpringBootConfiguration点开查看发现里面还是应用了@Configuration）
- @EnableAutoConfiguration
- @ComponentScan

###@Configuration

这个注解表示该类是一个java config，没什么好解释的，过。

###@ComponentScan

- **自定扫描路径下边带有@Controller，@Service，@Repository，@Component注解加入spring容器**
- **通过includeFilters加入扫描路径下没有以上注解的类加入spring容器**
- **通过excludeFilters过滤出不用加入spring容器的类**

###@EnableAutoConfiguration

这个注解是最为重要的，它的作用是根据类路径中的jar包、类自动生成配置bean，并加载IoC容器里面。这是SpringBoot简化配置的实现。

```java
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage  //自动扫描当前主程序类的 同级以及子级的包组件
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}
```

####@AutoConfigurationPackage

这个注解的作用是**自动扫描当前主程序类的同级以及子级的包组件**，比如（@Configuration和@Componet、@Service等），把它们注册到Ioc容器里面。**这也就是为什么，我们要把主类放在项目的最高级中**（好像和上面的@ComponentScan作用一样？不明白这里为什么还要这个注解）

#### @Import(EnableAutoConfigurationImportSelector.class)

最关键的要属@Import(EnableAutoConfigurationImportSelector.class)，借助**EnableAutoConfigurationImportSelector**，@EnableAutoConfiguration可以帮助SpringBoot应用将所有符合条件的@Configuration配置都加载到当前SpringBoot创建并使用的IoC容器。

####自动配置幕后英雄：SpringFactoriesLoader

借助于Spring框架原有的一个工具类：SpringFactoriesLoader的支持，@EnableAutoConfiguration可以智能的自动配置功效才得以大功告成！

SpringFactoriesLoader属于Spring框架私有的一种扩展方案，其主要功能就是从指定的配置文件META-INF/spring.factories加载配置。

```java
public abstract class SpringFactoriesLoader {
    //...
    public static <T> List<T> loadFactories(Class<T> factoryClass, ClassLoader classLoader) {
        ...
    }
    public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
        ....
    }
}
```

配合@EnableAutoConfiguration使用的话，它更多是提供一种配置查找的功能支持，即根据@EnableAutoConfiguration的完整类名org.springframework.boot.autoconfigure.EnableAutoConfiguration作为查找的Key,获取对应的一组@Configuration类

所以，@EnableAutoConfiguration自动配置的魔法骑士就变成了：**从classpath中搜寻所有的META-INF/spring.factories配置文件，并将其中org.springframework.boot.autoconfigure.EnableutoConfiguration对应的配置项通过反射（Java Refletion）实例化为对应的标注了@Configuration的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。**

##new SpringApplication(primarySources)

```java

public static ConfigurableApplicationContext run(Class<?>[] primarySources,
			String[] args) {
		return new SpringApplication(primarySources).run(args);
}
```

从代码上可以看出，调用了SpringApplication的静态方法run。这个run方法会构造一个SpringApplication的实例，然后再调用这里实例的run方法就表示启动SpringBoot。

创建SpringApplication做了下面这些事情：

1、配置source

2、判断是否是web程序（通过判断类加载器是否能加载到一些Spring的web类）

3、设置ApplicationContextInitializer

4、设置ApplicationListener

5、设置主类

```java
	/**
	 * Create a new {@link SpringApplication} instance. The application context will load
	 * beans from the specified primary sources (see {@link SpringApplication class-level}
	 * documentation for details. The instance can be customized before calling
	 * {@link #run(String...)}.
	 * @param resourceLoader the resource loader to use
	 * @param primarySources the primary bean sources
	 * @see #run(Class, String[])
	 * @see #setSources(Set)
	 */
	@SuppressWarnings({ "unchecked", "rawtypes" })
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        //判断是否是web程序
        //(javax.servlet.Servlet和
		//org.springframework.web.context.ConfigurableWebApplicationContext
        //都必须在类加载器中存在)，
        //并设置到webEnvironment属性中
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
     // 从spring.factories文件中找出key为ApplicationContextInitializer的类并
   //实例化后设置到SpringApplication的initializers属性中。这个过程也就是找出所有的应用程序初始化器
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
	  // 从spring.factories文件中找出key为ApplicationListener的类并实例化后设置到
       //SpringApplication的listeners属性中。这个过程就是找出所有的应用程序事件监听器
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        //找出main类，这里是DemoApplication类
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

###ApplicationContextInitializer是干嘛的？

```java
/*
在spring上下文（ConfigurableApplicationContext）刷新（refresh）之前调用。
通常用于需要一些编程初始化的Web应用程序中的applicationContext。 For example, registering property sources or activating profiles against the {@linkplain ConfigurableApplicationContext#getEnvironment() context's environment}.
*/
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {
	/**
	 * Initialize the given application context.
	 * @param applicationContext the application to configure
	 */
	void initialize(C applicationContext);
}
```

###SpringApplicationEvent

这里的应用程序事件(ApplicationEvent)有应用程序启动事件(ApplicationStartedEvent)，失败事件(ApplicationFailedEvent)，准备事件(ApplicationPreparedEvent)等。

## run()

```java
/**
	 * Run the Spring application, creating and refreshing a new
	 * {@link ApplicationContext}.
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return a running {@link ApplicationContext}
	 */
	public ConfigurableApplicationContext run(String... args) {
		//构造一个StopWatch，观察SpringAppication的启动过程
        StopWatch stopWatch = new StopWatch();
        //记录SpringApplication的启动时间
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		//设置系统属性为Headless模式，这个模式是在主机没有键盘、鼠标、显示器这些设备，却又想调用它们的方法的时候使用的
        configureHeadlessProperty();
        //启动SpringApplicationRunListeners，监听Springboot的run过程
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
            //初始化默认应用参数类
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
            //根据运行监听器和应用参数来准备 Spring 环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
            //打印springboot彩蛋
			Banner printedBanner = printBanner(environment);
			//根据程序类型创建特定的应用程序上下文
            context = createApplicationContext();
            //准备异常报告器
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			//准备应用上下文，看下面的源码
            prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
            //刷新应用上下文，主要就是refresh方法的执行
			refreshContext(context);
            //这个方法的源码是空的，目前可以做一些自定义的后置处理操作。
			afterRefresh(context, applicationArguments);
            //记录SpringApplication的停止计时时间
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
            //通知所有监听器context已经启动完成了
			listeners.started(context);
            //执行所有 Runner 运行器
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
            //触发所有 SpringApplicationRunListener 监听器的 running 事件方法
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
        //返回应用上下文
		return context;
	}
```

```javascript
private void prepareContext(ConfigurableApplicationContext context,
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments, Banner printedBanner) {
    // 10.1）绑定环境到上下文
    context.setEnvironment(environment);

    // 10.2）配置上下文的 bean 生成器及资源加载器
    postProcessApplicationContext(context);

    // 10.3）为上下文应用所有初始化器
    applyInitializers(context);

    // 10.4）触发所有 SpringApplicationRunListener 监听器的 contextPrepared 事件方法
    listeners.contextPrepared(context);

    // 10.5）记录启动日志
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }

    // 10.6）注册两个特殊的单例bean
    context.getBeanFactory().registerSingleton("springApplicationArguments",
            applicationArguments);
    if (printedBanner != null) {
        context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
    }

    // 10.7）加载所有资源
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[0]));

    // 10.8）触发所有 SpringApplicationRunListener 监听器的 contextLoaded 事件方法
    listeners.contextLoaded(context);
    
    
private void callRunners(ApplicationContext context, ApplicationArguments args) {
		List<Object> runners = new ArrayList<>();
		runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
		runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
		AnnotationAwareOrderComparator.sort(runners);
		for (Object runner : new LinkedHashSet<>(runners)) {
			if (runner instanceof ApplicationRunner) {
				callRunner((ApplicationRunner) runner, args);
			}
			if (runner instanceof CommandLineRunner) {
				callRunner((CommandLineRunner) runner, args);
			}
		}
	}    
```

### Runner是干嘛的？

如果你想要在Springboot启动完成的时候进行一些操作，比如打印一下信息，解析命令行中的信息，你可以实现Runner的run(..)方法。

```java
@Component
@Order(1) //用来表示执行顺序
public class TestApplicationRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("服务启动，TestApplicationRunner执行启动加载任务...");
        String[] sourceArgs = args.getSourceArgs();
        if(null != sourceArgs){
            for (String s : sourceArgs) {
                System.out.println(s);
            }
        }
    }
}
```
Runner分为两种，CommandLineRunner的参数是最原始的参数，没有做任何处理。ApplicationRunner的参数是ApplicationArguments，是对原始参数做了进一步的封装。

```java
@FunctionalInterface
public interface CommandLineRunner {

	/**
	 * Callback used to run the bean.
	 * @param args incoming main method arguments
	 * @throws Exception on error
	 */
	void run(String... args) throws Exception;

}
@FunctionalInterface
public interface ApplicationRunner {

	/**
	 * Callback used to run the bean.
	 * @param args incoming application arguments
	 * @throws Exception on error
	 */
	void run(ApplicationArguments args) throws Exception;

}
```





## 总结

SpringBoot启动的时候，不论调用什么方法，都会构造一个SpringApplication的实例，然后调用这个实例的run方法，这样就表示启动SpringBoot。

在run方法调用之前，也就是构造SpringApplication的时候会进行初始化的工作，初始化的时候会做以下几件事：

1. 把参数sources设置到SpringApplication属性中，这个sources可以是任何类型的参数。本文的例子中这个sources就是DemoApplication的class对象
2. 判断是否是web程序，并设置到webEnvironment这个boolean属性中
3. 找出所有的初始化器，默认有5个，设置到initializers属性中
4. 找出所有的应用程序监听器，默认有9个，设置到listeners属性中
5. 找出运行的主类(main class)

SpringApplication构造完成之后调用run方法，启动SpringApplication，run方法执行的时候会做以下几件事：

1. 构造一个StopWatch，观察SpringApplication的执行
2. 找出所有的SpringApplicationRunListener并封装到SpringApplicationRunListeners中，用于监听run方法的执行。监听的过程中会封装成事件并广播出去让初始化过程中产生的应用程序监听器进行监听
3. 构造Spring容器(ApplicationContext)，并返回
    3.1 创建Spring容器的判断是否是web环境，是的话构造AnnotationConfigEmbeddedWebApplicationContext，否则构造AnnotationConfigApplicationContext
    3.2 初始化过程中产生的初始化器在这个时候开始工作
    3.3 Spring容器的刷新(完成bean的解析、各种processor接口的执行、条件注解的解析等等)
4. 从Spring容器中找出ApplicationRunner和CommandLineRunner接口的实现类并排序后依次执行

![springboot启动过程](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/spring/springboot%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B.png)


