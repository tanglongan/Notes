SpringBoot在打包的时候会将依赖包也打进最终的Jar，变成一个可运行的FatJar。简单说会形成一个Jar in Jar的结构。

默认情况下，JDK提供的ClassLoader只能识别Jar中的class文件和加载classpath下的其他jar包中的class文件。对于在jar包中的jar包是无法加载的。

## 储备知识

### URLStreamHandler

Java中描述资源是使用URL。URL用于打开一个资源链接的方法是`java.net.URL#openConnection()`。由于URL用于表达各种各样的资源，打开资源的具体动作由`java.net.URLStreamHandler`这个类的子类来完成。根据不同的协议，会有不同的handler实现。而JDK内置了相当多的handler实现用于应对不同的协议。比如`jar`、`file`、`http`等等。URL内部有一个静态`HashTable`属性，用于保存已经被发现的协议和handler实例的映射。

获得URLStreamHandler有三种方法：

1. 实现URLStreamHandlerFactory接口，通过方法URL.setURLStreamHandlerFactory设置。该属性是一个静态属性，且只能被设置一次。
2. 直接提供URLStreamHandler的子类，作为URL的构造方法的入参之一。但是在JVM中有固定的规范要求：子类的类名必须是 Handler ，同时最后一级的包名必须是协议的名称。比如自定义了Http的协议实现，则类名必然为xx.http.Handler
3. JVM 启动的时候，需要设置java.protocol.handler.pkgs系统属性，如果有多个实现类，那么中间用 | 隔开。因为JVM在尝试寻找Handler时，会从这个属性中获取包名前缀，最终使用包名前缀.协议名.Handler，使用Class.forName方法尝试初始化类，如果初始化成功，则会使用该类的实现作为协议实现。

### Archive

SpringBoot定义了一个接口用于描述资源，也就是org.springframework.boot.loader.archive.Archive。该接口有两个实现，分别是org.springframework.boot.loader.archive.ExplodedArchive和org.springframework.boot.loader.archive.JarFileArchive。前者用于在文件夹目录下寻找资源，后者用于在jar包环境下寻找资源。而在SpringBoot打包的fatJar中，则是使用后者。

### 打包

SpringBoot提供了一个插件spring-boot-maven-plugin用于把程序打包成一个可执行的jar包。

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

打包完生成的executable-jar-1.0-SNAPSHOT.jar内部的结构如下：

```xml
├── META-INF
│   ├── MANIFEST.MF
│   └── maven
│       └── spring.study
│           └── executable-jar
│               ├── pom.properties
│               └── pom.xml
├── lib
│   ├── aopalliance-1.0.jar
│   ├── classmate-1.1.0.jar
│   ├── spring-boot-1.3.5.RELEASE.jar
│   ├── spring-boot-autoconfigure-1.3.5.RELEASE.jar
│   ├── ...
├── org
│   └── springframework
│       └── boot
│           └── loader
│               ├── ExecutableArchiveLauncher$1.class
│               ├── ...
└── spring
    └── study
        └── executablejar
            └── ExecutableJarApplication.class
```

然后可以直接执行jar包就能启动程序

```shell
java -jar executable-jar-1.0-SNAPSHOT.jar
```

打包出来fat jar内部有4种文件类型：

1. META-INF文件夹：程序入口，其中MANIFEST.MF用于描述jar包的信息
2. lib目录：放置第三方依赖的jar包，比如springboot的一些jar包
3. spring boot loader相关的代码
4. 模块自身的代码

MANIFEST.MF文件的内容

```
Manifest-Version: 1.0
Implementation-Title: executable-jar
Implementation-Version: 1.0-SNAPSHOT
Archiver-Version: Plexus Archiver
Built-By: Format
Start-Class: spring.study.executablejar.ExecutableJarApplication
Implementation-Vendor-Id: spring.study
Spring-Boot-Version: 1.3.5.RELEASE
Created-By: Apache Maven 3.2.3
Build-Jdk: 1.8.0_20
Implementation-Vendor: Pivotal Software, Inc.
Main-Class: org.springframework.boot.loader.JarLauncher
```

可以看到最后一行的Main-Class是org.springframework.boot.loader.JarLauncher，`当使用java -jar执行jar包的时候会调用JarLauncher的main方法，而不是我们编写的SpringApplication`。那么JarLauncher这个类是的作用是什么的？它是SpringBoot内部提供的工具Spring Boot Loader提供的一个用于执行Application类的工具类(fat jar内部有spring loader相关的代码就是因为这里用到了)。可以理解为Spring Boot Loader提供了一套标准用于执行SpringBoot打包出来的jar。



### SpringBoot Loader相关类

- Launcher：各种Launcher的基础抽象类，用于启动应用程序；跟Archive配合使用。目前有3个实现
    - JarLauncher
    - WarLauncher
    - PropertiesLauncher。
- Archive：归档文件的基础抽象类
    - JarFileArchive就是jar包文件的抽象。它提供了一些方法比如getUrl会返回这个Archive对应的URL；getManifest方法会获得Manifest数据等。
    - ExplodedArchive是文件目录的抽象。
- JarFile：对jar包的封装，每个JarFileArchive都会对应一个JarFile。JarFile被构造的时候会解析内部结构，去获取jar包里的各个文件或文件夹，这些文件或文件夹会被封装到Entry中，也存储在JarFileArchive中。如果Entry是个jar，会解析成JarFileArchive。

### JarLauncher的执行过程

JarLauncher的main方法：

```java
public static void main(String[] args) {
    // 构造JarLauncher，然后调用它的launch方法。参数是控制台传递的
    new JarLauncher().launch(args);
}
```

JarLauncher被构造的时候会调用父类ExecutableArchiveLauncher的构造方法。

ExecutableArchiveLauncher的构造方法内部会去构造Archive，这里构造了JarFileArchive。构造JarFileArchive的过程中还会构造很多东西。

JarLauncher的launch方法：

```java
protected void launch(String[] args) {
  try {
      
    // 在系统属性中设置注册了自定义的URL处理器：org.springframework.boot.loader.jar.Handler。
    // 如果URL中没有指定处理器，会去系统属性中查询
    JarFile.registerUrlProtocolHandler();
      
    //getClassPathArchives方法在会去找lib目录下对应的第三方依赖JarFileArchive，同时也会项目自身的JarFileArchive。
    //根据getClassPathArchives得到的JarFileArchive集合去创建类加载器ClassLoader。
    //这里会构造一个LaunchedURLClassLoader类加载器，这个类加载器继承URLClassLoader，并使用这些JarFileArchive集合的URL构造成URLClassPath
    //LaunchedURLClassLoader类加载器的父类加载器是当前执行类JarLauncher的类加载器
    ClassLoader classLoader = createClassLoader(getClassPathArchives());
      
    //getMainClass方法会去项目自身的Archive中的Manifest中找出key为Start-Class的类
    launch(args, getMainClass(), classLoader);
  }
  catch (Exception ex) {
    ex.printStackTrace();
    System.exit(1);
  }
}

// Archive的getMainClass方法
// 这里会找出spring.study.executablejar.ExecutableJarApplication这个类
public String getMainClass() throws Exception {
     Manifest manifest = getManifest();
     String mainClass = null;
     if (manifest != null) {
      		mainClass = manifest.getMainAttributes().getValue("Start-Class");
     }
     if (mainClass == null) {
			throw new IllegalStateException("No 'Start-Class' manifest entry specified in " + this);
     }
     return mainClass;
}

// launch重载方法
protected void launch(String[] args, String mainClass, ClassLoader classLoader) throws Exception {
      	// 创建一个MainMethodRunner，并把args和Start-Class传递给它
 		Runnable runner = createMainMethodRunner(mainClass, args, classLoader);
      	// 构造新线程
 		Thread runnerThread = new Thread(runner);
      	// 线程设置类加载器以及名字，然后启动
 		runnerThread.setContextClassLoader(classLoader);
 		runnerThread.setName(Thread.currentThread().getName());
 		runnerThread.start();
}
```

JarLauncher继承于org.springframework.boot.loader.ExecutableArchiveLauncher。

该类的无参构造方法最主要的功能就是构建了当前main方法所在的FatJar的JarFileArchive对象。下面来看launch方法。该方法主要是做了2个事情：

- 以FatJar为file作为入参，构造JarFileArchive对象。获取其中所有的资源目标，取得其Url，将这些URL作为参数，构建了一个URLClassLoader。
- 以第一步构建的ClassLoader加载MANIFEST.MF文件中Start-Class指向的业务类，并且执行静态方法main。进而启动整个程序。

通过静态方法org.springframework.boot.loader.JarLauncher#main就可以顺利启动整个程序。这里面的关键在于SpringBoot自定义的classLoader能够识别FatJar中的资源，包括有：在指定目录下的项目编译class、在指令目录下的项目依赖jar。JDK默认用于加载应用的AppClassLoader只能从jar的根目录开始加载class文件，并且也不支持jar in jar这种格式。

为了实现这个目标，SpringBoot首先从支持jar in jar中内容读取做了定制，也就是支持多个!/分隔符的url路径。SpringBoot定制了以下两个方面：

- 实现了一个java.net.URLStreamHandler的子类org.springframework.boot.loader.jar.Handler。该Handler支持识别多个!/分隔符，并且正确的打开URLConnection。打开的Connection是SpringBoot定制的org.springframework.boot.loader.jar.JarURLConnection实现。

- 实现了一个java.net.JarURLConnection的子类org.springframework.boot.loader.jar.JarURLConnection。该链接支持多个!/分隔符，并且自己实现了在这种情况下获取InputStream的方法。而为了能够在org.springframework.boot.loader.jar.JarURLConnection正确获取输入流，SpringBoot自定义了一套读取ZipFile的工具类和方法。这部分和ZIP压缩算法规范紧密相连，就不深入了。

能够读取多个!/的url后，事情就变得很简单了。上文提到的ExecutableArchiveLauncher的launch方法会以当前的FatJar构建一个JarFileArchive，并且通过该对象获取其内部所有的资源URL，这些URL包含项目编译class和依赖jar包。在构建这些URL的时候传入的就是SpringBoot定制的Handler。将获取的URL数组作为参数传递给自定义的ClassLoaderorg.springframework.boot.loader.LaunchedURLClassLoader。该ClassLoader继承自UrlClassLoader。UrlClassLoader加载class就是依靠初始参数传入的Url数组，并且尝试Url指向的资源中加载Class文件。有了自定义的Handler，再从Url中尝试获取资源就变得很容易了。SpringBoot自定义的ClassLoader就能够加载FatJar中的依赖包的class文件了。

