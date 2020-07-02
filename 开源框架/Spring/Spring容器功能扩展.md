> - ApplicationContext和BeanFactory都是用来加载Bean的。
>
> - ApplicationContext提供了更多的扩展功能，并且ApplicationContext包含了BeanFactory的所有功能。

### 1、容器初始化概览

先从ClassPathXmlApplicationContext作为切入点，开始对整体功能进行分析。

```java
/** Spring容器上下文的构造方法 */
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh) throws BeansException {
    this(configLocations, refresh, null);
}
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent){
    super(parent);
    setConfigLocations(configLocations);
    if (refresh) {
        refresh(); //核心方法
    }
}
```

配置文件以文件路径数组形式传入，ClassPathXmlApplicationContext可以对数组进行解析并加载。对于解析及功能实现都在refresh()方法实现。

```java
public void refresh() throws BeansException, IllegalStateException {

		synchronized (this.startupShutdownMonitor) {
			// 1、准备刷新上下文环境
			prepareRefresh();

			// 2、初始化BeanFactory，并进行XML文件读取
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// 3、对BeanFactory进行功能扩充
			prepareBeanFactory(beanFactory);

			try {
				//4、子类覆盖方法（留给子类扩展使用的回调方法）
				postProcessBeanFactory(beanFactory);

				// 5、激活各种BeanFactory处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				// 6、注册拦截Bean创建的Bean处理器，这里只是注册，真正的调用是在getBean()的时候
				registerBeanPostProcessors(beanFactory);

				// 7、初始化国际化消息
				initMessageSource();

				// 8、初始化当前应用上下文中的事件广播器
				initApplicationEventMulticaster();

				// 9、留给子类来初始化其他的Bean（Spring实现该方法来初始化Web容器）
				onRefresh();

				// 10、在所有注册的Bean中查找监听器Bean，注册到事件广播器中
				registerListeners();

				// 11、初始化所有的 singleton beans（非懒加载的Bean）
				finishBeanFactoryInitialization(beanFactory);

				// 12、完成刷新，并且发布或广播相应的事件
				finishRefresh();

			} catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + ex);
				}
				// 销毁已经创建的单例对象
				destroyBeans();
				//重新设置BeanFactory的激活属性active的值
				cancelRefresh(ex);
				throw ex;
			} finally {
				resetCommonCaches();
			}
		}
	}
```

总体概括一下refresh()方法的步骤

- 初始化前的准备工作。例如对系统属性或者环境变量进行准备及验证。

    在某种情况下项目的使用需要读取某些系统属性，而这个变量的设置可能会影响到系统的正确性。那么ClassPathXMLApplicationContext为我们提供了这个准备函数就非常有必要了，它可以在Spring容器启动的时候提前对必须的变量进行存在性验证。

- 初始化BeanFactory，并进行XML文件读取。

    由于ClassPathXmlApplicationContext包含了BeanFactory所提供的一切特征，那么这一步中将会复用BranFactory中配置文件读取解析以及其他功能，这一步之后ClassPathXmlApplicationContext实际上已经包含了BeanFactory所提供的功能。例如获取Bean的相关方法等。

- 对BeanFactory进行各种填充。

    @Qualifier和@Autowired是非常熟知的注解了，这两个注解正是在这一步中增加的支持。

- 子类覆盖方法做额外的处理。

- 激活各种BeanFactory的处理器。

- 注册拦截Bean创建的bean处理器。这里只是注册，真正调用是在getBean()的时候

- 为上下文初始化Message源，即对不同语言的消息体进行国际化处理

- 初始化应用消息广播器，并放入“applicationEventMulticaster”的Bean中。

- 留给子类来初始化其他Bean的onRefresh()方法

- 在所有注册的bean中查找listener bean，注册到消息广播器。

- 初始化剩下的非懒加载的单实例

- 完成刷新过程，通知生命周期处理器lifecycleProcessor刷新过程，同时发出ContextRefreshEvent通知别人。

### 2、环境变量准备

### 3、加载BeanFactory

obtainFreshBeanFactory()方法就是实现BeanFactory的地方，也就是这个函数执行完毕后ApplicationContext就具备了BeanFactory的全部功能。

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 初始化BeanFactory，并进行XML文件读取，将得到的BeanFactory记录在当前实体的属性中
    refreshBeanFactory();
    //返回当前实体的beanFactory属性
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}

protected final void refreshBeanFactory() throws BeansException {
    // refresh()方法可能被重复调用来重新初始化容器。如果已经存在BeanFactory了，就清理掉Bean，关闭现有的BeanFactory
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }

    try {
        // 创建DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 为BeanFactory设置序列化ID
        beanFactory.setSerializationId(getId());
        // 设置BeanFactory的两个属性(是否允许Bean的循环引用、是否允许同名不同定义的Bean、设置@Autowired和@Qulifier注解解析器)
        customizeBeanFactory(beanFactory);
        // 初始化DocumentReader，并进行XML文件读取和解析
        loadBeanDefinitions(beanFactory);
        //使用全局变量记录BeanFactory类实例
        this.beanFactory = beanFactory;
    } catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```

### 4、功能扩展

进入函数prepareBeanFactory之前，Spring已经完成 了配置的解析，而ApplicationContext在功能上的扩展也由此展开。

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //1、设置beanFactory的classLoader为当前Context的classLoader
    beanFactory.setBeanClassLoader(getClassLoader());
    //2、设置SpEL表达式解析器（#{bean.xxx}的解析）
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    //3、属性编辑器。对bean的属性等设置管理的一个工具
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
	//4、添加BeanPostProcessor
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    //5、设置几个忽略自动装配的接口
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
	//6、设置几个自动装配的特殊规则
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
    //7、增加对AspectJ的支持
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 8、添加默认的系统属性Bean
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

上面几个重要步骤，主要进行了几个方面的扩展

- 增加SpEL语言的支持
- 增加对属性编辑器的支持
- 增加一些内置类，比如EnvironmentAware、MessageSourceAware的信息注入
- 设置了依赖功能可忽略的接口
- 注册了一些固定依赖的属性
- 增加了AspectJ的支持
- 将相关环境变量及属性注册以单例模式注册

#### 4.1、增加SpEL的扩展



#### 4.2、增加属性注册编辑器

#### 4.3、添加ApplicationContextAwareProcessor处理器

#### 4.4、设置忽略依赖

#### 4.5、注册依赖





