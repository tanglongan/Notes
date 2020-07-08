在使用Spring时，可能由于设计不佳或者各种因素，导致类之间相互依赖。这些类可能单独使用时不会出问题，但是在使用Spring进行管理的时候可能就会抛出BeanCurrentlyInCreationException等异常 。当抛出这种异常时表示Spring解决不了该循环依赖。

## 1、循环依赖的产生

循环依赖产生可能有多种可能：

- A的构造方法中依赖了B的实例对象，同时B的构造方法中也依赖了A的实例对象
- A的构造方法中依赖了B的实例对象，同时B的某个filed或setter需要A的实例对象，反之亦然。
- A的某个filed或setter需要B的实例对象，同时B的某个filed或setter需要A的实例对象，反之亦然。

这三种情况，只有第一种是Spring也不能解决的，后面两种情况解决了。

Spring中解决循环依赖是有前提条件：**针对scope单例并且没有显式指明不需要解决循环依赖的对象，而且要求该对象没有被代理过**。

## 2、循环依赖的解决

Spring循环依赖解决的理论依据是**Java基于引用传递，当我们获取到对象的引用时，对象的field或属性是可以延后赋值的**。

Spring单例对象的初始化过程分为三个步骤：

- 实例化createBeanInstance()：实际上就是调用对应的构造方法创建对象，此时只是调用构造方法，对象刚刚产生，属性都换没初始化赋值。
- 填充属性populateBean()：这一步是对对象的属性进行赋值。
- 初始化initializeBean()：调用bean定义时指定的init方法或者AfterPropertiesSet方法。（spring.xml中<bean>标签可以指定初始化方法，这些方法书要是对默认属性值进行覆盖）

### 2.1、三级缓存

在Spring容器的生命周期内，一个单例对象有且仅有一个，因此这个对象就应该是在缓存中，Spring大量使用了Cache的手段，循环依赖中也使用了。

```java
/** 已完成所有初始化过程的单例对象缓存池 */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(256);
/** 单例对象工厂的缓存池 */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
/** 提前曝光的单例对象缓存池 */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
```

### 2.2、解决方法

首先在创建Bean的方法开始，调用了getSingletonBean()方法先从缓存池中获取，获取到就直接返回。

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return (singletonObject != NULL_OBJECT ? singletonObject : null);
}
```

首先解释两个关键的判断方法

- **isSingletonCurrentlyInCreation(beanName)**：判断对应的单例对象是否在创建中，当单例对象没有被初始化完全(例如A定义的构造函数依赖了B对象，得先去创建B对象，或者在populatebean过程中依赖了B对象，得先去创建B对象，此时A处于创建中)。
- **allowEarlyReference**：是否允许从singletonFactories中通过getObject拿到对象

然后分析getSingleton()的整个过程，Spring首先从singletonObjects（一级缓存）中尝试获取，如果获取不到并且对象在创建中，则尝试从earlySingletonObjects(二级缓存)中获取，如果还是获取不到并且允许从singletonFactories通过getObject获取，则通过singletonFactory.getObject()(三级缓存)获取。如果获取到了则移除对应的singletonFactory，将singletonObject放入到earlySingletonObjects，其实就是将三级缓存提升到二级缓存中！

```java
this.earlySingletonObjects.put(beanName, singletonObject);
this.singletonFactories.remove(beanName);
```

Spring解决循环依赖的诀窍就在于singletonFactories这个cache，这个cache中存的是类型为ObjectFactory，其定义如下：

```java
public interface ObjectFactory<T> {
    T getObject() throws BeansException;
}
```

在bean创建过程中，有两处比较重要的匿名内部类实现了该接口。一处是

```java
new ObjectFactory<Object>() {
    public Object getObject() throws BeansException {
        try {
            return createBean(beanName, mbd, args);
        } catch (BeansException ex) {
            destroySingleton(beanName);
            throw ex;
        }    
    }
}
```

另一处是

```java
addSingletonFactory(beanName, new ObjectFactory<Object>() {
    public Object getObject() throws BeansException {
        return getEarlyBeanReference(beanName, mbd, bean);
    }
   }
);
```

此处就是解决循环依赖的关键，这段代码发生在createBeanInstance之后，也就是说单例对象此时已经被创建出来的。这个对象已经被生产出来了，虽然还不完美（还没有进行初始化的第二步和第三步），但是已经能被人认出来了（根据对象引用能定位到堆中的对象），所以Spring此时将这个对象提前曝光出来让大家认识，让大家使用。

这样做有什么好处呢？

用一个案例分析“A的某个field或者setter依赖了B的实例对象，同时B的某个field或者setter依赖了A的实例对象”这种情况下的循环依赖，步骤如下：

- A首先完成了初始化第一步，并且将自己提前曝光在了singletonFactories缓存中了
- A然后准备第二步填充属性了，发现自己依赖对象B，此时尝试去getBean(B)，发现B还没有创建，所以要开始自己的创建流程
- B完成了初始化第一步，将自己提前曝光在了singletonFactories缓存中
- B准备填充属性的时候，发现自己依赖对象A，于是尝试从缓存中获取，尝试一级缓存singletonObjects(肯定没有，因为A还没初始化完全)，尝试二级缓存earlySingletonObjects（也没有），尝试三级缓存singletonFactories。由于A通过ObjectFactory将自己提前曝光了，所以B能够通过ObjectFactory.getObject拿到A对象(虽然A还没有初始化完全，但是总比没有好呀)，B拿到A对象后顺利完成了初始化阶段1、2、3，完全初始化之后将自己放入到一级缓存singletonObjects中。
- B的创建流程结束了，回到了A的创建getBean(B)的过程了，A此时能拿到B的对象顺利完成自己的初始化阶段2、3，最终A也完成了初始化，长大成人，进去了一级缓存singletonObjects中，而且更加幸运的是，由于B拿到了A的对象引用，所以B现在hold住的A对象也蜕变完美了！

## 3、循环依赖的总结

1. 开始创建初始化A
2. 之后“非完美对象A”开始填充属性字段，此时发现需要一个B的对象。同时已标记A处于正在初始化阶段。
3. 显然接下来，开始去初始化B的对象，同样的手法，到设置属性阶段，发现需要A对象 
4. 于是乎，spring又开始去初始化对象A的依赖，此时先从缓存singletonObjects去取，没有再去看是否正处于初始阶段，是则再从缓存earlySingletonObjects中取，再没有，则看是否存在allowEarlyReference，是则从singletonFactories中取 
5. 将早期对象A设置到B中，再把B设置到A中。

## 4、常见问题

### 如何解决属性注入循环依赖？

- 创建A，先从缓存取->缓存没有(首次肯定没有)->getSingleton创建A，这里会把A放到一个正在创建集合表示A正在创建(code1)，然后执行构造方法，构造执行完毕之后，会将自身放到缓存暴露早期引用(code2)，然后再属性填充发现需要依赖B，因此去创建B。

- 创建B的流程和A一模一样，也是走缓存，最后也会在属性填充的时候发现自己依赖A，然后就去创建A，等于又回到了创建A的逻辑

- 欣慰的是这一次A已经可以在缓存中获取了，得到了A(这个A其实还是不完整的)，B就可以顺序初始化，B初始化好了，A也就可以初始化好了，由于对象在整个容器中是一个，最后A初始化完成之后，B所依赖的A也是一个完整的A了，完美解决。

    ```java
    //DefaultSingletonBeanRegistry
    beforeSingletonCreation(beanName);
    
    //code1
    protected void beforeSingletonCreation(String beanName) {
    		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
    		throw new BeanCurrentlyInCreationException(beanName);
    	}
    }
    
    //code2
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +"' to allow for resolving potential circular references");
        }
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }
    
    
    ```

### 为什么不能解决构造函数依赖注入?

- 首先创建A，缓存没有，将自己添加到正在创建的集合，然后构造的时候发现自己依赖B，因此转向获取B，关键在这里，此时A还未来得及将自己放到缓存。
- 然后获取B，也是一样，在构造的那一步发现自己依赖A，因此去获取A。
- 到这应该清楚了，缓存中获取不到A，因此需要走创建流程，而创建流程会将自己添加到正在创建的集合里面去，而A在之前已经添加过一次，这一次就会报错，如code1所示，下面的添加会返回false,因此会抛出BeanCurrentlyInCreationException异常。

### 为什么不能解决非单例依赖注入？

- 非单例的每次创建都是new一个对象，IOC不会对其进行缓存，如果AB循环依赖，那么创建A1的时候依赖B1，然后要创建B1的时候发现依赖A，由此再去创建A2，然后A2有需要一个B2，一直下去没有尽头。不过Spring在代码中是有判断的，在AbstractBeanFactory的doGetBean中，如果缓存中获取Bean失败，就会判断需要获取的Bean是否为处于创建中的原型Bean，如果是就会抛出异常。

- 这里就是检查原型Bena的循环依赖并且抛出异常的地方，在2.3中的堆栈日志也能够看到异常抛出的位置。

### 为什么有三级缓存不是二级缓存？

- 二级缓存可以解决循环依赖问题吗？其实是可以解决循环依赖问题的。使用三级缓存的目的是将返回早期引用对象的这个动作用于预留一个扩展接口，在三级缓存中获取bean调用的getEarlyBeanReference会回调SmartInstinationBeanPostProcessor()方法，开发者可以在这个方法中对返回的Bean做修改，如果AB依赖，B就会调用getEarlyBeanReference()方法获取A，从而B会从三级缓存中拿到A，并移动到二级缓存。
- 换个角度，假如只有2级缓存，那么用户扩展的动作就要放在二级缓存来做，这样每一获取的时候，以先从一级缓存中获取，获取不到就二级缓存执行一次singletonFactory.getObject()触发一下获取Bean，这样这个方法可能或被执行多次，每一次创建一个新的对象。这样肯定会有问题了，引入三级缓存之后，singletonFactory.getObject()方法只会在第一次获取的时候调用，前两级都不存在才会调用，调用完毕之后将自己放入到二级缓存中，并从三级缓存中移除，后面每一次获取都是拿的这一次调用生成的对象。
- 如果不需要扩展，放进去的早期引用就是不完整的Bean，不考虑给其他后置处理器扩展，这样使用2级缓存是可以的。A直接将自己放入二级缓存，循环依赖的时候B直接从二级缓存获取A的早期引用保证B初始化成功再将自己加到一级缓存，然后A初始化成功后添加到一级缓存，但是这样就不能扩展了，因为这个扩展点很重要，在AOP的AnnotationAwareAspectJAutoProxyCreator就通过这个扩展点来保证代理对象的返回和代理对象的循环依赖问题解决(保证循环依赖的时候，返回的对象也是代理对象)。

### 如果一个代理对象没有被循环依赖？

- 一个对象A如果被B循环依赖，并且A是一个代理对象，那么Spring不仅解决了循环依赖并且可以保证依赖的也是代理对象，如果A是一个代理对象，但是没有被循环依赖，比如A就是一个纯粹被增强了的Bean会怎么样？

- 如果A代理对象没有被循环依赖，那么他的getEarlyBeanReference就不会被调用(这个方法是在B循环依赖A并访问三级缓存的时候触发的)，因此ProxyA会在AnnotationAwareAspectJAutoProxyCreator的postProcessAfterInitialization方法中，返回代理对象。因此在初始化完毕之后的循环依赖检查的时候，发现二级缓存中是没有的，因此会直接返回前面postProcessAfterInitialization创建的代理对象。

    