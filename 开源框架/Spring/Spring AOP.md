

> XML是Spring的基础。尽管Spring一再简化配置，并且大量使用注解取代XML配置方式。但是还是XML还是Spring的基础。
>
> 要在Spring中开启AOP功能，需要在XML中配置<aop:aspectj-autoproxy>

Spring是否支持注解的AOP是由一个配置文件控制的，也就是`<aop:aspectj-autoproxy>`。在容器初始化过程中加载配置文件的时候，一旦遇到aspectj-autoproxy注解时就会使用解析器AspectJAutoProxyBeanDefinitionParser进行解析，Spring AOP也正是从这个类实现的。

### 1、注册AnnotatoinAwareAspectJAutoProxyCreator

所有解析器都是对BeanDefinitionParser接口的统一实现，入口都是从parse()方法开始的，其方法如下

```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
    // 注册AnnotatoinAwareAspectJAutoProxyCreator
    AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
    //对于注解中子类的实现
    extendBeanDefinition(element, parserContext);
    return null;
}
```

最关心的是registerAspectJAnnotationAutoProxyCreatorIfNecessary()方法逻辑

```java
public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(ParserContext parserContext, Element sourceElement) {
    //注册或升级AutoProxyCreator定义beanName为InternalAutoProxyCreator的BeanDefinition
    BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(
        parserContext.getRegistry(), parserContext.extractSource(sourceElement));
    //对于proxy-target-class以及expose-proxy的处理
    useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
    //注册组件并通知，便于监听器做进一步的处理
    registerComponentIfNecessary(beanDefinition, parserContext);
}
```

#### 1.1、注册或升级AnnotatoinAwareAspectJAutoProxyCreator

对于AOP的实现，基本上都是靠AnnotatoinAwareAspectJAutoProxyCreator去完成的，它可以根据@Point注解定义的切点来自动代理相匹配的bean。但是为了配置方便，Spring使用了自定义配置来帮助我们自动注册AnnotatoinAwareAspectJAutoProxyCreator，其注册过程如下：

```java
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(
    BeanDefinitionRegistry registry, @Nullable Object source) {
    return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}

private static BeanDefinition registerOrEscalateApcAsRequired(
    								Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    //如果已经存在了自动代理创建器，且存在的自动代理创建器与现在不一致，那么需要根据优先级来判断需要使用哪个
    if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
        BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            if (currentPriority < requiredPriority) {
                //改变bean最重要的就是改变bean所对应的className属性
                apcDefinition.setBeanClassName(cls.getName());
            }
        }
        //存在且相同，就无需创建直接返回
        return null;
    }
	
    RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
    beanDefinition.setSource(source);
    beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
    beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
    return beanDefinition;
}
```

#### 1.2、处理proxy-target-class以及expose-proxy属性

useClassProxyingIfNecessary实现了proxy-target-class属性以及expose-proxy属性的处理

```java
private static void useClassProxyingIfNecessary(BeanDefinitionRegistry registry, @Nullable Element sourceElement) {
    if (sourceElement != null) {
        //proxy-target-class属性处理（Spring AOP使用JDK动态代理还是CGLIB生成代理）
        boolean proxyTargetClass = Boolean.parseBoolean(sourceElement.getAttribute(PROXY_TARGET_CLASS_ATTRIBUTE));
        if (proxyTargetClass) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        }
        //对于expose-Proxy属性的判断
        boolean exposeProxy = Boolean.parseBoolean(sourceElement.getAttribute(EXPOSE_PROXY_ATTRIBUTE));
        if (exposeProxy) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
        }
    }
}
```

- 建议使用JDK动态代理，因为CGLIB代理无法对final修饰的方法进行代理，因为它们不能被覆写
- 强制使用CGLIB代理需要将`<aop:config>的proxy-target-class属性设置为true`。

#### 1.3、动态代理

- JDK代理：其代理对象必须是某个接口的实现，它是通过在运行期间创建一个接口的实现类来完成对目标对象的代理。
- CGLIB代理：实现原理类似于JDK动态代理，只是它在运行期间生成代理对象是针对目标类扩展的子类。CGLIB是高效的代码生成包，底层是依靠ASM（开源的字节码编辑类库）操作字节码实现的，性能比JDK强。

