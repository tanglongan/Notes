### 典型使用方式

#### 基于XML的方式

```java
BeanFactory context = new ClassPathXmlApplicationContext("applicationContext.xml");
AppService bean = context.getBean(AppService.class);
```

#### 基于Java Config

```java
BeanFactory applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
AppService appService = applicationContext.getBean(AppService.class);
```

ClassPathXmlApplicationContext和AnnotationConfigApplicationContext都是Spring上下文的实现类，容器的初始化的过程也都是在它们的构造方法中进行的。构造方法源码分别如下：

```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    this();
    register(annotatedClasses);
    refresh();		//初始化过程的核心方法
}

public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
    this(new String[]{configLocation}, true, null);
}
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent){
    super(parent);
    setConfigLocations(configLocations);
    if (refresh) {
        refresh(); //初始化过程的核心方法
    }
}
```

可以看到不管是基于XML还是Java Config的方式使用Spring，容器上下文在初始化的时候，最终都是通过refresh()方法实现的。

### refresh方法

由于ClassPathXmlApplicationContext和AnnotationConfigApplicationContext都是继承自AbstractApplicationContext抽象类的，refresh()方法的大部分实现逻辑就在这个抽象类中。

```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1、初始化之前的准备（1、清理容器上下文中的缓存、2、处理配置文件的占位符以及校验配置文件相关属性）
        prepareRefresh();

        // 2、读取配置信息并解析为一个个BeanDefinition的bean定义信息，将bean定义信息注册到BeanFactory中
        // 注：bean还没初始化，只是将其定义信息提取出来了，注册也只是将这些信息都保存到了注册中心(beanName-> beanDefinition的map)
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 3、设置应用上下文属性（上下文类加载器和BeanPostProcessor处理器），同时配置和注册了几个特殊的bean
        prepareBeanFactory(beanFactory);

        try {
            // 4、留给子类的扩展点，回调方法。到这里，所有的Bean都加载并注册完成了，但是都还没有初始化
            // Bean如果实现了BeanFactoryPostProcessor接口，那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory方法
            postProcessBeanFactory(beanFactory);

            // 5、调用BeanFactoryPostProcessor各个实现类的 postProcessBeanFactory(factory) 方法
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6、注册BeanPostProcessor的实现类
            registerBeanPostProcessors(beanFactory);

            // 7、国际化消息的初始化
            initMessageSource();

            // 8、初始化当前应用上下文中的事件广播器
            initApplicationEventMulticaster();

            // 9、这里是一个回调方法，留给子类去实现。像SpringMVC/Boot实现了这个方法来创建Tomcat/Jetty这样的Web容器
            onRefresh();

            // 10、注册事件监听器，监听器需要实现 ApplicationListener 接口。
            registerListeners();

            // 11、初始化所有的 singleton beans（lazy-init的除外）
            finishBeanFactoryInitialization(beanFactory);

            // 12、发布或广播相应的事件
            finishRefresh();

        } catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + ex);
            }
            // 销毁已经创建的单例对象
            destroyBeans();
            // 重新设置BeanFactory的激活属性active的值
            cancelRefresh(ex);
            throw ex;
        } finally {
            resetCommonCaches();
        }
    }
}
```

可以看到refresh方法中有按照顺序有12步操作。

