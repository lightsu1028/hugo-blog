+++
title = "Spring源码解析BeanDefinition解析"
date = "2023-01-19"
description = "Spring BeanDefinition源码解析"
tags = [
    "Spring",
    "BeanDefinition"
]
categories = [
    "Spring"
]
image = "bd.jpg"
draft=false
+++

主要是介绍Spring framwork中的BeanDefinition模块的源码解析。
<!--more-->

## 前言
在Spring中我们可以通过编程式的方式显示声明定义BeanDefinition：
```java
// 创建一个Spring容器
AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
// 获取容器中名为userService的BeanDefinition
BeanDefinition userService = applicationContext.getBeanDefinition("userService");
System.out.println(userService);
```

## ClassPathBeanDefinitionScanner
### 扫描器是什么
>bean定义扫描解析器用于在classpath下发现候选的bean,使用指定的**BeanFactory**或者**ApplicationContext**注册bean定义`BeanDefinition`。

上面是官方源码中的注释，其实写的很清楚了，扫描器通过入参classpath使用容器进行bean定义的注册，使用相关的filter进行过滤注册，默认过滤器会对包含如下注解的类进行扫描解析:
* `org.springframework.stereotype.Component @Component`
* `org.springframework.stereotype.Repository @Repository`
* `org.springframework.stereotype.Service @Service`
* `org.springframework.stereotype.Con--troller @Controller`

看下相关的构造函数API：
```java
// 构造函数中传入容器
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry) {
        // 使用默认过滤器
        this(registry, true);
}

// 构造函数传入容器和是否使用默认过滤器
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters) {
    this(registry, useDefaultFilters, getOrCreateEnvironment(registry));
}
```
扫描器根据指定的包路径比如在@ComponentScan注解中指定，通过资源解析器`ResourcePatternResolver`扫描该路径下的class文件,最终通过元数据读取器`MetadataReader`解析成一个一个的BeanDefinition注册到容器上下文中去，解析时候可以设置相应的规则filter,比如设置哪些class文件不需要解析又有哪些需要。

### 扫描解析入口方法
Spring上下文容器在扫描时会调用`org.springframework.context.annotation.ClassPathBeanDefinitionScanner#scan`进行解析指定包路径下面的候选
定义，具体Spring是在哪调用该方法进入扫描解析流程的后续再详细分析，本篇专注解析逻辑本身。
```java
public int scan(String... basePackages) {
    // 获取已经扫描的bean定义个数
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
    // 开始扫描解析
    doScan(basePackages);
    
    // Register annotation config processors, if necessary.
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }
    // 计算本次实际扫描解析到的bean定义个数
    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
```
scan方法中并没有核心的解析逻辑，只是计算了下本次解析的BeanDefinition个数，具体的解析逻辑位于doScan(basePackages)方法中。

## ResourceLoader
```java
// 读取文件资源
AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
Resource resource = ctx.getResource("file://D:\\githu\\spring-framework-5.3.10\\tuling\\src\\main\\java\\com\\demo\\service\\UserService.java");
System.out.println(resource.contentLength());
System.out.println(resource.getFilename());

// 读取网络资源
Resource resource1 = ctx.getResource("https://www.baidu.com");
System.out.println(resource1.contentLength());
System.out.println(resource1.getURL());

// 读取类路径资源
Resource resource2 = ctx.getResource("classpath:spring.xml");
System.out.println(resource2.contentLength());
System.out.println(resource2.getURL());
```
Spring中上下文容器AbstractApplicationContext由于继承了`DefaultResourceLoader`，DefaultResourceLoader实现了`ResourceLoader`，所以可以使用容器上下文应对不同的资源路径获取不同的`Resource`。
而在ClassPathBeanDefinitionScanner内部最终则会使用`org.springframework.core.io.support.PathMatchingResourcePatternResolver#getResources`解析包路径下得到文件资源。

![](PathMatchingResourcePatternResolver.png) 
PathMatchingResourcePatternResolver通过实现ResourcePatternResolver接口最终实现ResourceLoader的能力。

## MetadataReader
在Spring中需要去解析类的信息，比如类名、类中的方法、类上的注解，这些都可以称之为类的元数据，所以Spring中对类的元数据做了抽象，并提供了一些工具类。
MetadataReader表示类的元数据读取器，默认实现类为SimpleMetadataReader。比如：
```java
SimpleMetadataReaderFactory simpleMetadataReaderFactory = new SimpleMetadataReaderFactory();

// 构造一个MetadataReader
MetadataReader metadataReader = simpleMetadataReaderFactory.getMetadataReader("com.demo.service.UserService");

// 得到一个ClassMetadata，并获取了类名
ClassMetadata classMetadata = metadataReader.getClassMetadata();
System.out.println(classMetadata.getClassName());

// 获取一个AnnotationMetadata，并获取类上的注解信息
AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
// 类上是否包含@Component注解 可以递归检查
System.out.println(annotationMetadata.hasMetaAnnotation(Component.class.getName()));
for (String annotationType : annotationMetadata.getAnnotationTypes()) {
	System.out.println(annotationType);
}
```
## ExcludeFilter和IncludeFilter

