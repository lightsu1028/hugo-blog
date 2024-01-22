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
* `org.springframework.stereotype.Controller @Controller`

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
扫描器根据指定的包路径比如使用@ComponentScan注解指定，通过资源解析器`ResourcePatternResolver`扫描该路径下的class文件,最终通过元数据读取器`MetadataReader`解析成一个一个的BeanDefinition注册到容器上下文中去，解析时候可以设置相应的规则filter,比如设置哪些class文件不需要解析又有哪些需要。

### 扫描解析入口方法
Spring上下文容器在扫描时会调用`org.springframework.context.annotation.ClassPathBeanDefinitionScanner#scan`进行解析指定包路径下面的候选bean定义，具体Spring是在哪调用该方法进入扫描解析流程的后续再详细分析，本篇专注解析逻辑本身。
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

### Spring如何注册扫描器
知道了扫描器具体是干什么的后，Spring是在哪些地方去初始化解析器的呢？
