Dubbo实现要点

- Provider模块：提供API、实现API、暴露服务(启动Tomcat、nettyServer)、服务本地注册、服务注册中心注册

- Consumer模块：通过接口名从注册中心获取服务地址、调用服务

- Registry模块：保存服务配置信息(服务名：List<URL>)

- RPCProtocol模块：基于Tomcat模块的HttpProtocol、基于Netty的DubboProtocol

- Framework模块：框架实现



Java SPI

- java spi不能单独的获取某个指定的实现类
- java spi没有ioc和aop机制





