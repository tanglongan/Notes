# MyBatis3源码深度解析

#### 第1章 搭建Mybatis源码环境

​	    1.1 Mybatis3简介
​	    1.2 环境准备
​		 1.3 获取Mybatis源码
​        1.4 导入Mybatis源码到IDE
​        1.5 HSQLDB数据库简介

#### 第2章 JDBC规范详解

​        2.1 JDBC API简介
​            2.1.1 建立数据源连接
​            2.1.2 执行SQL语句
​            2.1.3 处理SQL执行结果
​            2.1.4 使用JDBC操作数据库
​        2.2 JDBC API中的类与接口
​            2.2.1 java.sql包详解
​            2.2.2 javax.sql包详解
​        2.3 Connection详解
​            2.3.1 JDBC驱动类型
​            2.3.2 java.sql.Driver接口
​            2.3.3 Java SPI机制简介
​            2.3.4 java.sql.DriverAction接口
​            2.3.5 java.sql.DriverManager类
​            2.3.5 javax.sql.DataSource接口
​            2.3.6 使用JNDI API增强应用可移植性
​            2.3.7 关闭Connection对象
​        2.4 Statement详解
​            2.4.1 java.sql.Statement接口
​            2.4.2 java.sql.PreparedStatement接口
​            2.4.3 java.sql.CallableStatement接口
​            2.4.4 获取自增长的键值
​        2.5 ResultSet详解
​            2.5.1 ResultSet类型
​            2.5.2 ResultSet并行性
​            2.5.3 ResultSet可保持性
​            2.5.4 ResultSet属性设置
​            2.5.5 ResultSet游标移动
​            2.5.6 修改ResultSet对象
​            2.5.7 关闭ResultSet对象
​        2.6 DatabaseMetaData详解
​            2.6.1 创建DatabaseMetaData对象
​            2.6.2 获取数据源基本信息
​            2.6.3 获取数据源支持特性
​            2.6.5 获取数据源限制
​            2.6.7 获取SQL对象及属性
​            2.6.8 获取事务支持
​        2.7 JDBC事务
​            2.7.1 事务边界与自动提交
​            2.7.2 事务隔离级别
​            2.7.3 事务中保存点

#### 第3章 Mybatis常用工具类

​        3.1 使用SQL类生成语句
​        3.2 使用ScriptRunner执行脚本
​        3.3 使用SqlRunner操作数据库
​        3.4 MetaObject详解
​        3.5 MetaClass详解
​        3.6 ObjectFactory详解
​        3.7 ProxyFactory详解

#### 第4章 Mybatis核心组件介绍

​        4.1 使用Mybatis操作数据库
​        4.2 Mybatis核心组件
​        4.3 Configuration详解
​        4.4 Executor详解
​        4.5 MappedStatement详解
​        4.6 StatementHandler详解
​        4.7 TypeHandler详解
​        4.9 ParameterHandler详解
​        4.9 ResultSetHandler详解

#### 第5章 SqlSession创建过程

​        5.1 XPath方式解析XML文件
​        5.2 Configuration实例创建过程
​        5.3 SqlSession实例创建过程

#### 第6章 SqlSession执行Mapper过程

​        6.1 Mapper接口注册过程
​        6.2 MappedStatement注册过程
​        6.3 Mapper方法调用过程详解
​        6.4 SqlSession执行Mapper过程

#### 第7章 Mybatis缓存

​        7.1 Mybatis缓存的使用 139
​        7.2 Mybatis缓存实现类 140
​        7.3 Mybatis一级缓存实现原理
​        7.4 Mybatis二级缓存实现原理
​        7.5 Mybatis使用Redis缓存
第8章 Mybatis日志实现 155
​        8.1 Java日志体系 155
​        8.2 Mybatis日志实现 159
​        8.3 本章小结 165
第9章 动态SQL实现原理 166
​        9.1 动态SQL的使用 166
​        9.2 SqlSource与BoundSql详解 168
​        9.3 LanguageDriver详解 171
​        9.4 SqlNode详解 174
​        9.5 动态SQL解析过程 179
​        9.6 从源码角度分析#{}和${}区别 189
​        9.7 本章小结 193
第10章 Mybatis插件实现原理 194
​        10.1 Mybatis插件实现原理 194
​        10.2 自定义一个分页插件 203
​        10.3 自定义慢SQL统计插件 211
​        10.4 本章小结 212
第11章 Mybatis级联映射与懒加载 214
​        11.1 Mybatis级联映射详解 214
​            11.1.1 准备工作 214
​            11.1.2 一对多关联映射 217
​            11.1.3 一对一关联映射 219
​            11.1.4 Discriminator详解 221
​        11.2 Mybatis懒加载机制 223
​        11.3 Mybatis级联映射实现原理 224
​            11.3.1 ResultMap详解 224
​            11.3.2 ResultMap解析过程 225
​            11.3.3 级联映射实现原理 231
​        11.4 懒加载实现原理 238
​        11.5 本章小结 243
第12章 Mybatis与Spring整合案例 245
​        12.1 准备工作 245
​        12.2 Mybatis与Spring整合 246
​        12.3 用户注册案例 248
​        12.4 本章小结 251
第13章 Mybatis Spring实现原理 252
​        13.1 Spring中的一些概念 252
​        13.2 Spring容器启动过程 255
​        13.3 Mapper动态代理对象注册过程 256
​        13.4 Mybatis整合Spring事务管理 260
​        13.5 本章小结 264