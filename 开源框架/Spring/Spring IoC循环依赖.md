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