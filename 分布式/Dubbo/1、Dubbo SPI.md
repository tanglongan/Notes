## 1、Dubbo SPI

SPI(Service Provider Interface)，是一种服务发现机制，Dubbo的SPI是从Java SPI增强而来，Dubbo的文档中给了三个增强的理由：

> - JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
> - 增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。
> - 如果扩展点加载失败，连扩展点的名称都拿不到了。比如：JDK 标准的 ScriptEngine，通过 `getName()` 获取脚本类型的名称，但如果 RubyScriptEngine 因为所依赖的 jruby.jar 不存在，导致 RubyScriptEngine 类加载失败，这个失败原因被吃掉了，和 ruby 对应不起来，当用户执行 ruby 脚本时，会报不支持 ruby，而不是真正失败的原因。

对于一个开源的RPC框架，可扩展性是十分重要的，那么Dubbo是怎么来实现这一点的呢，我们可以在Dubbo的源码中随处可见到类似下面这样的代码：

```
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

这个ExtensionLoader就是Dubbo扩展能力的基础，也是理解Dubbo运行机制的基石，那么下面来了解和分析SPI是什么。先看一个Dubbo SPI的案例

```java
/** 定义一个扩展点的接口 */
@SPI("udp") 
public interface Transporter {
    void send(String msg);
    @Adaptive("transporter")
    void send(String msg, URL url);
}

/**实现类一*/
public class UDPTransporter implements Transporter{
    public void send(String msg) {
        System.out.println("Transfer " + msg + " thorough UDP");
    }
    public void send(String msg, URL url) {
        send(msg);
    }
}

/**实现类二*/
@Activate(value = "reliability")
public class TCPTransporter implements Transporter{
    public void send(String msg) {
        System.out.println("Transfer " + msg + " thorough TCP");
    }
    public void send(String msg, URL url) {
        send(msg);
    }
}
```

关联SPI机制，在/resources/META-INF/dubbo目录下创建如下文件

![image-20191217231726993.png](.images/1.png)

配置文件内容：

```properties
tcp=com.wanglaomo.playground.dubbo.provider.spi.TCPTransporter
udp=com.wanglaomo.playground.dubbo.provider.spi.UDPTransporter
```

相比于Java的SPI机制，我们可以看到Dubbo的SPI多了一层映射关系{tcp -> TCPTransporter, udp -> UDPTransporter}。测试代码如下：

```java
ExtensionLoader extensionLoader = ExtensionLoader.getExtensionLoader(Transporter.class);

System.out.println(">>>>>>>>> 基础用法");
Transporter transporter = (Transporter) extensionLoader.getExtension("tcp");
transporter.send("msg");

System.out.println(">>>>>>>>> 默认实现");
Transporter defaultTransporter = (Transporter) extensionLoader.getDefaultExtension();
defaultTransporter.send("msg");

System.out.println(">>>>>>>>> 自适应实现");
Transporter adaptiveTransporter = (Transporter) extensionLoader.getAdaptiveExtension();
adaptiveTransporter.send("msg", URL.valueOf("test://localhost/test?transporter=udp"));

System.out.println(">>>>>>>>> 自动激活实现");
List<Transporter> activeTransporters = extensionLoader.getActivateExtension(URL.valueOf("test://localhost/test?reliability=true"), (String) null);
for(Transporter activateTransporter : activeTransporters) {
    activateTransporter.send("msg");
}
```

测试输出结果如下：

```
>>>>>>>>> 基础用法
Transfer msg thorough TCP
>>>>>>>>> 默认实现
Transfer msg thorough UDP
>>>>>>>>> 自适应实现
Transfer msg thorough TCP
>>>>>>>>> 自动激活实现
Transfer msg thorough TCP
```

在这个案例里面我们一次性地把Dubbo SPI的所有用法都过了个遍，在Dubbo源码中也无外乎就这几种。

## 2、源码分析

Dubbo SPI使用案例

```java
public void test() {
    ExtensionLoader loader = ExtensionLoader.getExtensionLoader(Protocol.class);
    Protocol protocol = (Protocol) loader.getExtension("dubbo");
}
```

### 2.1、获取扩展加载器

```java
// 自适应扩展点（有且仅有一个）
ExtensionLoader loader = ExtensionLoader.getExtensionLoader(Protocol.class);
```

ExtensionLoader工厂类中关键属性

```java

Map<String, Object> cachedActivates = new ConcurrentHashMap<>(); 					// 可激活的ExtensionLoader
Class<?> cachedAdaptiveClass;														// 自适应的扩展点实现（有且只有一个）
String cachedDefaultName; 															// 默认实现的名称 
ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>(); 	// 所有的扩展点别名和扩展点实例的集合
```

通过这4个成员变量，可以看出ExtensionLoader是一个很重的工厂类对象，再结合下面会讲到的扩展点自动注入，ExtensionLoader基本上实现了一个功能完备的扩展点IOC工厂。进入ExtensionLoader#getExtensionLoader方法之后

```java
/**
  * 创建ExtensionLoader实例(loader1)，此时 loader1的type=Protocol.class，
  * 在为loader1的objectFactory 属性赋值的时候，触发ExtensionFactory对应的ExtensionLoader实例化；
  */
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null) {
        throw new IllegalArgumentException("Extension type == null");
    }
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
    }
    if (!withExtensionAnnotation(type)) {
        String msg = "Extension type("+type+") is'not an extension,it's not annotated with @"+SPI.class.getSimpleName()+ ";
        throw new IllegalArgumentException(msg);
    }

    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

首先做一些扩展点接口的参数判断，然后通过ConcurrentHashMap#putIfAbsent方法来实现ExtensionLoader的延迟加载。然后可以看一下ExtensionLoader的创建过程，如下代码：

```java
private ExtensionLoader(Class<?> type) {
    this.type = type;
    //创建ExtensionLoader实例(loader2)，此时 loader2 的 type=ExtensionFactory.class，objectFactory = null;
    objectFactory = (type == ExtensionFactory.class ? null : 
                     ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}
```

可以看到基本上这里只是个工厂类的壳子，相应的扩展类并没有被加载，因为扩展点实现类的加载也是延迟的。至于这个ExtensionFactory，因为它本身也被标注为@SPI扩展点，在获取它的ExtensionLoader时也会来这里走一遭，所以在这里先进行ExtensionFactory的扩展点加载和初始化操作。上述代码执行流程会执行ExtensionFactory对应的ExtensionLoader的getAdaptiveExtension方法

```java
// 获取自适应扩展实例对象
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError != null) {
            throw new IllegalStateException("Failed to create adaptive instance");
        }
		// 双重检查机制确保创建唯一
        synchronized (cachedAdaptiveInstance) {
            //先从缓存中获取。第一次会进入下面的创建并保存缓存的过程；以后都是直接从缓存获取返回即可
            instance = cachedAdaptiveInstance.get();
            if (instance == null) {
                try {
                    instance = createAdaptiveExtension();
                    cachedAdaptiveInstance.set(instance);
                } catch (Throwable t) {
                    createAdaptiveInstanceError = t;
                    throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                }
            }
        }
    }
    return (T) instance;
}

//创建自适应扩展点的实例对象
private T createAdaptiveExtension() {
    try {
        //1、先通过getAdaptiveExtensionClass方法获取自适应扩展点的实现类的Class
        //2、通过Class的newInstance方法实例化扩展实现类的实例对象
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}

// 获取当前扩展点的自适应扩展点实现类(标记了@Adaptive注解的那个实现类，只能有一个接口的实现类被它标记)的Class
private Class<?> getAdaptiveExtensionClass() {
    //加载所有扩展点实现类
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}

// 加载所有扩展点实现类
private Map<String, Class<?>> getExtensionClasses() {
    //先从缓存中获取。第一次初始化的时候会进入下面的加载过程；以后都是直接在缓存中返回
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        // 双重检查机制，避免重复加载
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
                // 搜索并加载所有扩展点实现类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

// 在系统约定的目录中读取相关的SPI配置文件
//Dubbo中约定的SPI配置目录有
private Map<String, Class<?>> loadExtensionClasses() {
    //从扩展点接口@SPI注解中获取默认扩展点实现
    cacheDefaultExtensionName();
    Map<String, Class<?>> extensionClasses = new HashMap<>();
	
    for (LoadingStrategy strategy : strategies) {
        loadDirectory(extensionClasses, 
                      strategy.directory(), 
                      type.getName(), 
                      strategy.preferExtensionClassLoader(), 
                      strategy.excludedPackages());
        
        loadDirectory(extensionClasses, 
                      strategy.directory(), 
                      type.getName().replace("org.apache", "com.alibaba"), 
                      strategy.preferExtensionClassLoader(), 
                      strategy.excludedPackages());
    }

    return extensionClasses;
}

//获取默认扩展点实现，并将默认扩展点名称保存在cachedDefaultName属性中
private void cacheDefaultExtensionName() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation == null) {
        return;
    }
    
    String value = defaultAnnotation.value();
    if ((value = value.trim()).length() > 0) {
        String[] names = NAME_SEPARATOR.split(value);
        if (names.length > 1) {
            throw new IllegalStateException("More than 1 default extension name on extension");
        }
        if (names.length == 1) {
            cachedDefaultName = names[0];
        }
    }
}

//获取类加载器，加载所有Dubbo SPI的配置文件，读取并创建扩展点实现类的Class对象
private void loadDirectory(Map<String, Class<?>> extensionClasses, 
                           String dir, 
                           String type,
                           boolean extensionLoaderClassLoaderFirst, 
                           String... excludedPackages) {
    String fileName = dir + type;
    try {
        Enumeration<java.net.URL> urls = null;
        ClassLoader classLoader = findClassLoader();

        //这里加载资源，类加载器有一个顺序
        //当前类的类加载器 ---> 加载ExtensionLoader类的类加载器 ---> 系统类加载器(Application ClassLoader)
        if (extensionLoaderClassLoaderFirst) {
            ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
            if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                urls = extensionLoaderClassLoader.getResources(fileName);
            }
        }

        if(urls == null || !urls.hasMoreElements()) {
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
        }

        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
                loadResource(extensionClasses, classLoader, resourceURL, excludedPackages);
            }
        }
    } catch (Throwable t) {
        logger.error(t);
    }
}

/**
  * 读取SPI配置文件
  */
private void loadResource(Map<String, Class<?>> extensionClasses,
                          ClassLoader classLoader,
                          java.net.URL resourceURL, 
                          String... excludedPackages) {
    try {
        try (InputStreamReader input = new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8)) {
            BufferedReader reader = new BufferedReader(input)
            String line;
            while ((line = reader.readLine()) != null) {
                // 下面几行是为了去除或忽略行内注释
                final int ci = line.indexOf('#');
                if (ci >= 0) {
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                        int i = line.indexOf('=');
                        if (i > 0) {
                            name = line.substring(0, i).trim();
                            line = line.substring(i + 1).trim();
                        }
                        if (line.length() > 0 && !isExcluded(line, excludedPackages)) {
                            //解析扩展点的实现类（生成对应的Class，并缓存起来，起到懒加载的作用）
                            loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException("Failed to load extension class", t);
                        exceptions.put(line, e);
                    }
                }
            }
        }
    } catch (Throwable t) {
        logger.error(t);
    }
}


/**
  * 加载扩展点实现类，并按照分类缓存起来
  * （1）普通扩展点实现
  * （2）自适应扩展点实现(有且仅有1个)，缓存在Class<?> cachedAdaptiveClass中
  * （3）默认扩展点实现(0个或1个)
  * （3）可激活扩展点实现，缓存在Map<String, Object> cachedActivates
  * （4）Wrapper包装扩展点，缓存在Set<Class<?>> cachedWrapperClasses
  */
private void loadClass(Map<String, Class<?>> extensionClasses, 
                       java.net.URL resourceURL, 
                       Class<?> clazz, 
                       String name) throws NoSuchMethodException {
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class.");
    }
    //当@Adaptive被标注在类上时，则表明Dubbo直接使用该类作为已实现自适应扩展的类，而不用Dubbo再自行生成。
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        cacheAdaptiveClass(clazz);
    } else if (isWrapperClass(clazz)) {
        cacheWrapperClass(clazz);
    } else {
        clazz.getConstructor();
        if (StringUtils.isEmpty(name)) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName());
            }
        }

        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) {
            cacheActivateClass(clazz, names[0]);
            for (String n : names) {
                cacheName(clazz, n);
                saveInExtensionClass(extensionClasses, clazz, n);
            }
        }
    }
}
```

Dubbo SPI实现了简单的IOC功能，类似于Spring IOC的，上面调用过程中创建自适应扩展点对象createAdaptiveExtension方法中调用了injectExtension()方法进行IOC注入

```java
private T injectExtension(T instance) {
    if (objectFactory == null) {
        return instance;
    }
    try {
        for (Method method : instance.getClass().getMethods()) {
            //只通过setterXXX方法进行注入，比较简单的实现
            if (!isSetter(method)) {
                continue;
            }
            // 标记@DisableInject注解的属性不注入
            if (method.getAnnotation(DisableInject.class) != null) {
                continue;
            }
            // 基本类型的参数就直接跳过
            Class<?> pt = method.getParameterTypes()[0];
            if (ReflectUtils.isPrimitives(pt)) {
                continue;
            }
            try {
                String property = getSetterProperty(method);
                Object object = objectFactory.getExtension(pt, property);
                if (object != null) {
                    method.invoke(instance, object);
                }
            } catch (Exception e) {
                logger.error("Failed to inject via method " + method.getName(), e);
            }

        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

### 2.2、获取扩展点实现

使用案例中的第二行调用

```java
Protocol protocol = (Protocol) loader.getExtension("dubbo");
```

```java
public T getExtension(String name) {
    if (StringUtils.isEmpty(name)) {
        throw new IllegalArgumentException("Extension name == null");
    }
    // 默认扩展点实现
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    // 先从缓存中获取扩展点实现
    final Holder<Object> holder = getOrCreateHolder(name);
    Object instance = holder.get();
    if (instance == null) {
        // 双重检查，保证扩展点的实例是单例的
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}

/**
  * 创建扩展点实例
  */
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            // 直接创建扩展点实例，并放入缓存中
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //自动注入扩展点的属性（IOC）
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        //循环包装自动扩展点对象（AOP）
        if (CollectionUtils.isNotEmpty(wrapperClasses)) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        initExtension(instance);
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException(t);
    }
}

```

