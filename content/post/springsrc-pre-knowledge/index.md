+++
title = "Spring源码解析前置知识"
date = "2023-01-11"
description = "spring源码解析前置知识点"
tags = [
    "Spring"
]
categories = [
    "Spring"
]
image = "spring-framework.png"
draft=false
+++

主要是介绍spring源码中涉及到的一些常用知识点例如一些API的用法介绍，便于读者在阅读后续Spring主流程源码时熟悉这些API。
<!--more-->

## BeanDefinition
`BeanDefinition`表示Bean定义，`BeanDefinition`中存在很多属性用来描述一个Bean的特点。比如：
* class，表示Bean类型
* scope，表示Bean作用域，单例或原型等
* lazyInit：表示Bean是否是懒加载
* initMethodName：表示Bean初始化时要执行的方法
* destroyMethodName：表示Bean销毁时要执行的方法
* ...

在Spring中，我们经常会通过以下几种方式来定义Bean:
* \<bean/>
* @Bean
* @Component(@Service,@Controller)

这些，我们可以称之***申明式***定义Bean。
还可以编程式定义Bean，那就是直接通过`BeanDefinition`，比如：
```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

// 生成一个BeanDefinition对象，并设置beanClass为User.class，并注册到ApplicationContext中
AbstractBeanDefinition beanDefinition = BeanDefinitionBuilder.genericBeanDefinition().getBeanDefinition();
beanDefinition.setBeanClass(User.class);
context.registerBeanDefinition("user", beanDefinition);

System.out.println(context.getBean("user"));
```
我们还可以通过BeanDefinition设置一个Bean的其他属性
```java
beanDefinition.setScope("prototype"); // 设置作用域
beanDefinition.setInitMethodName("init"); // 设置初始化方法
beanDefinition.setLazyInit(true); // 设置懒加载
```
和申明式事务、编程式事务类似，通过\<bean/>，@Bean，@Component等申明式方式所定义的Bean，最终都会被Spring解析为对应的`BeanDefinition`对象，并放入Spring容器中。
## BeanDefinitionReader
接下来，我们来介绍几种在Spring源码中所提供的`BeanDefinition`读取器（`BeanDefinitionReader`），这些`BeanDefinitionReader`在我们使用Spring时用得少，但在Spring源码中用得多，相当于Spring源码的基础设施。
### AnnotatedBeanDefinitionReader
可以直接把某个类转换为BeanDefinition，并且会解析该类上的注解，比如:
```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

AnnotatedBeanDefinitionReader annotatedBeanDefinitionReader = new AnnotatedBeanDefinitionReader(context);

// 将User.class解析为BeanDefinition
annotatedBeanDefinitionReader.register(User.class);

System.out.println(context.getBean("user"));
```
注意：它能解析的注解是：@Conditional，@Scope、@Lazy、@Primary、@DependsOn、@Role、@Description
### XmlBeanDefinitionReader
可以解析\<bean/>标签
```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

XmlBeanDefinitionReader xmlBeanDefinitionReader = new XmlBeanDefinitionReader(context);
int i = xmlBeanDefinitionReader.loadBeanDefinitions("spring.xml");

System.out.println(context.getBean("user"));
```
### ClassPathBeanDefinitionScanner
ClassPathBeanDefinitionScanner是扫描器，但是它的作用和BeanDefinitionReader类似，它可以进行扫描，扫描某个包路径，对扫描到的类进行解析，比如，扫描到的类上如果存在@Component注解，那么就会把这个类解析为一个BeanDefinition，比如：
```java
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
context.refresh();

ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(context);
scanner.scan("com.zhouyu");

System.out.println(context.getBean("userService"));
```
## BeanFactory
## ApplicationContext
### AnnotationConfigApplicationContext
### ClassPathXmlApplicationContext
### 国际化
### 资源加载
### 获取运行时环境
### 事件发布
## 类型转化
### PropertyEditor
### ConversionService
### TypeConverter
## OrderComparator
## BeanPostProcessor
## BeanFactoryPostProcessor
## FactoryBean
## ExcludeFilter和IncludeFilter
## MetadataReader、ClassMetadata、AnnotationMetadata