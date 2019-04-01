# Spring IOC

## BeanFactory

BeanFactory接口定义了Ioc容器最基本的功能。

| Modifier and Type       | Method and Description                                       |
| ----------------------- | ------------------------------------------------------------ |
| `boolean`               | `containsBean(String name)`Does this bean factory contain a bean definition or externally registered singleton instance with the given name? |
| `String[]`              | `getAliases(String name)`Return the aliases for the given bean name, if any. |
| `<T> T`                 | `getBean(Class<T> requiredType)`Return the bean instance that uniquely matches the given object type, if any. |
| `<T> T`                 | `getBean(Class<T> requiredType, Object... args)`Return an instance, which may be shared or independent, of the specified bean. |
| `Object`                | `getBean(String name)`Return an instance, which may be shared or independent, of the specified bean. |
| `<T> T`                 | `getBean(String name, Class<T> requiredType)`Return an instance, which may be shared or independent, of the specified bean. |
| `Object`                | `getBean(String name, Object... args)`Return an instance, which may be shared or independent, of the specified bean. |
| `<T> ObjectProvider<T>` | `getBeanProvider(Class<T> requiredType)`Return an provider for the specified bean, allowing for lazy on-demand retrieval of instances, including availability and uniqueness options. |
| `<T> ObjectProvider<T>` | `getBeanProvider(ResolvableType requiredType)`Return an provider for the specified bean, allowing for lazy on-demand retrieval of instances, including availability and uniqueness options. |
| `Class<?>`              | `getType(String name)`Determine the type of the bean with the given name. |
| `boolean`               | `isPrototype(String name)`Is this bean a prototype? That is, will [`getBean(java.lang.String)`](https://docs.spring.io/spring/docs/5.2.0.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/BeanFactory.html#getBean-java.lang.String-) always return independent instances? |
| `boolean`               | `isSingleton(String name)`Is this bean a shared singleton? That is, will [`getBean(java.lang.String)`](https://docs.spring.io/spring/docs/5.2.0.BUILD-SNAPSHOT/javadoc-api/org/springframework/beans/factory/BeanFactory.html#getBean-java.lang.String-) always return the same instance? |
| `boolean`               | `isTypeMatch(String name, Class<?> typeToMatch)`Check whether the bean with the given name matches the specified type. |
| `boolean`               | `isTypeMatch(String name, ResolvableType typeToMatch)`Check whether the bean with the given name matches the specified type. |

编程式使用Ioc容器的过程

```java
ClassPathResource res = new ClassPathResource("beans.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);
```

这样我们就可以通过factory对象来使用DefaultListableBeanFactory这个Ioc容器了，在使用Ioc容器时，需要如下几个步骤：

- 创建Ioc配置文件的抽象资源，这个抽象资源包含了BeanDefinition的定义信息。
- 创建一个BeanFactory，这里使用DefaultListableBeanFactory。
- 创建一个载入BeanDefinition的读取器，这里使用XmlBeanDefinitionReader来载入XML文件形式的BeanDefinition，通过一个回调配置给BeanFactory
- 从定义好的资源位置读入配置信息，具体的解析过程由XmlBeanDefinitionReader类完成，完成整个载入和注册Bean定义之后，需要的Ioc容器就建立起来的。这时候就可以直接使用Ioc容器了



## ApplicationContext

ApplicationContext在BeanFactory的基础上添加了许多功能：

- 支持不同的信息源，我们看到ApplicationContext扩展了MessageSource接口，这些信息源的扩展功能可以支持国际化的实现，为开发多语言版本的应用提供服务。
- 访问资源。这一特性体现在对ResourceLoader和Resource的支持上，这样我们可以从不同地方的得到bean定义资源。
- 支持应用事件，继承了接口ApplicationEventPublisher，从而在上下文中引入了事件机制。
- 在ApplicationContext中提供附加服务。

####Application容器的设计原理

以FileSystemXmlApplication为例子

作为一个具体的应用上下文，只需实现和它自身设计有关的两个功能

一个功能是，如果应用直接使用FileSystemXmlApplicationContext，对于实例化这个应用程序上下文的支持，同时启动Ioc容器的refresh()的过程。

```java
public FileSystemXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```

这个refresh()过程会牵涉Ioc容器启动的一系列复杂操作。

另一个功能是与FileSystemXmlApplcationContext设计具体相关的功能，这部分与怎样从文件系统中加载Xml的Bean资源定位有关。

```java
protected Resource getResourceByPath(String path) {
		if (path.startsWith("/")) {
			path = path.substring(1);
		}
		return new FileSystemResource(path);
	}

```



##IOC容器初始化过程

简单来说，Ioc容器的初始化是由前面的refresh()方法来启动的，具体来说，这个启动包括BeanDefinition的Resource定位、载入和注册三个基本过程。

第一个过程是Resource定位。这个Resouce定位指的是BeanDefinition的资源定位，它由ResourceLoader通过统一的getResource()接口来完成，这个Resource对各种形式的BeanDefinition的使用都提供了统一接口。常见的Resource有FileSystemResouce、ClassPathResource等。

第二个过程是BeanDifinition的载入。这个载入过程是把用户定义好的Bean表示成Ioc容器内部的数据结构，即BeanDifinition。

Bean definition contains the information called **configuration metadata**, which is needed for the container to know the following −

- How to create a bean
- Bean's lifecycle details
- Bean's dependencies

第三个过程是向Ioc容器注册这些BeanDefinition的过程。这个过程是通过调用BeanDefinitionRegistry接口的实现来完成的。这个注册过程把载入过程中解析得到的BeanDefiniiton向Ioc容器进行注册。注册到HashMap中去。



### BeanDefinition的Resource定位

Resource的定位由refresh()这个方法触发，refresh()调用refreshBeanFactory()，refreshBeanFactory()调用loadBeanDifinition(),loadBeanDifinition()调用getResourceLoader()得到DefaultResourceLoader,再调用ResourceLoader的getResource()方法，getResource()调用每个ApplicationContext自己实现的getResourceByPath()方法，最终得到Resource[]数组。

```java
public void refresh() throws BeansException, IllegalStateException {
        Object var1 = this.startupShutdownMonitor;
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            //这个方法触发了refreshBeanFactory()
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }

                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }


    protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        this.refreshBeanFactory();
        return this.getBeanFactory();
    }

protected final void refreshBeanFactory() throws BeansException {
        if (this.hasBeanFactory()) {
            this.destroyBeans();
            this.closeBeanFactory();
        }

        try {
            DefaultListableBeanFactory beanFactory = this.createBeanFactory();
            beanFactory.setSerializationId(this.getId());
            this.customizeBeanFactory(beanFactory);
            //这个方法完成了Ressource的定位和BeanDefinition的载入
            this.loadBeanDefinitions(beanFactory);
            Object var2 = this.beanFactoryMonitor;
            synchronized(this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        } catch (IOException var5) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var5);
        }
    }


protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    //得到这个ApplicationContext特定的Reader
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
        this.initBeanDefinitionReader(beanDefinitionReader);
    //进一步得到Resource，以获得Bedifinition
        this.loadBeanDefinitions(beanDefinitionReader);
    }


protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    
        Resource[] configResources = this.getConfigResources();
        if (configResources != null) {
            //由Reader去获取Resource
            reader.loadBeanDefinitions(configResources);
        }

        String[] configLocations = this.getConfigLocations();
        if (configLocations != null) {
            reader.loadBeanDefinitions(configLocations);
        }

    }

```

### BeanDefinition的解析和加载

接着前面的代码，BeanDefinition的解析要先解析Resource得到Document，再从Document中解析出BeanDefinition。

```java
//最后就是得到Resource的InputStream读取文件
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
        Assert.notNull(encodedResource, "EncodedResource must not be null");
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Loading XML bean definitions from " + encodedResource);
        }

        Set<EncodedResource> currentResources = (Set)this.resourcesCurrentlyBeingLoaded.get();
        if (currentResources == null) {
            currentResources = new HashSet(4);
            this.resourcesCurrentlyBeingLoaded.set(currentResources);
        }

        if (!((Set)currentResources).add(encodedResource)) {
            throw new BeanDefinitionStoreException("Detected cyclic loading of " + encodedResource + " - check your import definitions!");
        } else {
            int var5;
            try {
                InputStream inputStream = encodedResource.getResource().getInputStream();

                try {
                    InputSource inputSource = new InputSource(inputStream);
                    if (encodedResource.getEncoding() != null) {
                        inputSource.setEncoding(encodedResource.getEncoding());
                    }

                    var5 = this.doLoadBeanDefinitions(inputSource, encodedResource.getResource());
                } finally {
                    inputStream.close();
                }
            } catch (IOException var15) {
                throw new BeanDefinitionStoreException("IOException parsing XML document from " + encodedResource.getResource(), var15);
            } finally {
                ((Set)currentResources).remove(encodedResource);
                if (((Set)currentResources).isEmpty()) {
                    this.resourcesCurrentlyBeingLoaded.remove();
                }

            }

            return var5;
        }
    }

//通过Resource有可以得到XML解析出来的Document
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) throws BeanDefinitionStoreException {
        try {
            Document doc = this.doLoadDocument(inputSource, resource);
            int count = this.registerBeanDefinitions(doc, resource);
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Loaded " + count + " bean definitions from " + resource);
            }

            return count;
        } catch (BeanDefinitionStoreException var5) {
            throw var5;
        } catch (SAXParseException var6) {
            throw new XmlBeanDefinitionStoreException(resource.getDescription(), "Line " + var6.getLineNumber() + " in XML document from " + resource + " is invalid", var6);
        } catch (SAXException var7) {
            throw new XmlBeanDefinitionStoreException(resource.getDescription(), "XML document from " + resource + " is invalid", var7);
        } catch (ParserConfigurationException var8) {
            throw new BeanDefinitionStoreException(resource.getDescription(), "Parser configuration exception parsing XML from " + resource, var8);
        } catch (IOException var9) {
            throw new BeanDefinitionStoreException(resource.getDescription(), "IOException parsing XML document from " + resource, var9);
        } catch (Throwable var10) {
            throw new BeanDefinitionStoreException(resource.getDescription(), "Unexpected exception parsing XML document from " + resource, var10);
        }
    }

    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        BeanDefinitionDocumentReader documentReader = this.createBeanDefinitionDocumentReader();
        
        int countBefore = this.getRegistry().getBeanDefinitionCount();
        //解析document，得到BeanDefinition
        documentReader.registerBeanDefinitions(doc, this.createReaderContext(resource));
        return this.getRegistry().getBeanDefinitionCount() - countBefore;
    }

//按照Bean的规则解析Document中的Element
protected void doRegisterBeanDefinitions(Element root) {
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = this.createDelegate(this.getReaderContext(), root, parent);
        if (this.delegate.isDefaultNamespace(root)) {
            String profileSpec = root.getAttribute("profile");
            if (StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, ",; ");
                if (!this.getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    if (this.logger.isDebugEnabled()) {
                        this.logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec + "] not matching: " + this.getReaderContext().getResource());
                    }

                    return;
                }
            }
        }

        this.preProcessXml(root);
        this.parseBeanDefinitions(root, this.delegate);
        this.postProcessXml(root);
        this.delegate = parent;
    }
//主要由BeanDefinitionParserDelegate来完成
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();

            for(int i = 0; i < nl.getLength(); ++i) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element)node;
                    if (delegate.isDefaultNamespace(ele)) {
                        this.parseDefaultElement(ele, delegate);
                    } else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        } else {
            delegate.parseCustomElement(root);
        }

    }

```

接下来是对各种属性的解析，包括list、map、set、pros还有protory，构造函数等都放到BeanDefintion中。



###BeanDefinition的注册

前面我们得到的BeanDefinition需要注册到Ioc容器中，具体实现在DefaultListableBeanFactory的registerBeanDefinition()方法中

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) throws BeanDefinitionStoreException {
        Assert.hasText(beanName, "Bean name must not be empty");
        Assert.notNull(beanDefinition, "BeanDefinition must not be null");
        if (beanDefinition instanceof AbstractBeanDefinition) {
            try {
                ((AbstractBeanDefinition)beanDefinition).validate();
            } catch (BeanDefinitionValidationException var9) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName, "Validation of bean definition failed", var9);
            }
        }

        BeanDefinition existingDefinition = (BeanDefinition)this.beanDefinitionMap.get(beanName);
    //这个如果已经存在这个name的BeanDefinition，又不允许覆盖，就会报错
        if (existingDefinition != null) {
            if (!this.isAllowBeanDefinitionOverriding()) {
                throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
            }

            if (existingDefinition.getRole() < beanDefinition.getRole()) {
                if (this.logger.isInfoEnabled()) {
                    this.logger.info("Overriding user-defined bean definition for bean '" + beanName + "' with a framework-generated bean definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
                }
            } else if (!beanDefinition.equals(existingDefinition)) {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Overriding bean definition for bean '" + beanName + "' with a different definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
                }
            } else if (this.logger.isTraceEnabled()) {
                this.logger.trace("Overriding bean definition for bean '" + beanName + "' with an equivalent definition: replacing [" + existingDefinition + "] with [" + beanDefinition + "]");
            }

            this.beanDefinitionMap.put(beanName, beanDefinition);
        } else {
            if (this.hasBeanCreationStarted()) {
                Map var4 = this.beanDefinitionMap;
                synchronized(this.beanDefinitionMap) {
                    this.beanDefinitionMap.put(beanName, beanDefinition);
                    List<String> updatedDefinitions = new ArrayList(this.beanDefinitionNames.size() + 1);
                    updatedDefinitions.addAll(this.beanDefinitionNames);
                    updatedDefinitions.add(beanName);
                    this.beanDefinitionNames = updatedDefinitions;
                    if (this.manualSingletonNames.contains(beanName)) {
                        Set<String> updatedSingletons = new LinkedHashSet(this.manualSingletonNames);
                        updatedSingletons.remove(beanName);
                        this.manualSingletonNames = updatedSingletons;
                    }
                }
            } else {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                this.beanDefinitionNames.add(beanName);
                this.manualSingletonNames.remove(beanName);
            }

            this.frozenBeanDefinitionNames = null;
        }

        if (existingDefinition != null || this.containsSingleton(beanName)) {
            this.resetBeanDefinition(beanName);
        }

    }
```

##Bean的实例化和依赖注入

getBean()->doGetBean()->createBean()->instantiate()->populateBean->applyPropertyValues()->resolveReference()



这篇文章解决了我长久以来的一些疑惑，比如为什么通过构造方法注入的时候，发生循环依赖会报错？
因为在实例化Bean的时候通过构造方法实例的时候会判断参数中的类型，如果是BeanReference就调用getBean去实例化它，这时候就会发生死循环，导致Spring启动失败。

那为什么通过Setter方法的循环依赖就可以呢？

其实这个情况下，只有其中一个Bean是单例的Bean才可以，如果两个都是Prototype的也会报错。

因为对于单例Bean，在进行Setter依赖注入前，会先把实例化得到的单例Bean放到SingleTon的Map里面。Setter注入的时候直接可以从Map里面得到。

https://www.cnblogs.com/leeSmall/p/10228771.html

最后总结如下图：

![getbean](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/spring/getbean.png)