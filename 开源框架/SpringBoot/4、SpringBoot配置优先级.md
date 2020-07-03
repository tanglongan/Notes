### 1、默认配置文件目录

spring boot 启动会扫描以下位置的application.properties 或者application.yml文件作为spring boot 的默认配置文件 ，加载的优先由上到下，加载的时候，会把以下路径的文件都加载一遍。不同的配置内容会全部加载到系统，对于重复的配置内容，优先级别高的配置文件内容会覆盖优先级别低的配置文件内容。

| 路径              | 说明                                         |
| ----------------- | -------------------------------------------- |
| file:./config     | 工程目录下的config目录                       |
| file:/            | 工程目录，如果是maven项目，则是和pom.xml同级 |
| classpath:/config | 工程classpath路径下的config目录              |
| classpath:/       | 工程classpath路径下                          |

除了上述目录之外，还可以用`spring.config.locations`参数指定配置文件路径。

### 2、外部配置

有时候工程已经打成jar了 ，想修改系统的配置，SpringBoot也可以从jar包外面设置参数，加载配置； 以下设置优先级从高到低；高优先级的配置覆盖低优先级的配置，所有的配置会 形成互补配置。

- 命令行参数

    所有的配置都可以在命令行上进行指定 ，多个配置用空格分开； --配置项=值。

    java -jar spring-boot-02-config-02-0.0.1-SNAPSHOT.jar --server.port=8087 --server.context-path=/abc

- Java系统属性（System.getProperties()）。

- 操作系统环境变量。

- RandomValuePropertySource配置的random.*属性值。



