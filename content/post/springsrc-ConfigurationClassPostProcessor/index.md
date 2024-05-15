+++
title = "Spring ConfigurationClassPostProcessor"
date = "2024-05-04"
description = "Spring BeanDefinition解析源码"
tags = [
    "Spring",
    "BeanDefinition",
    "BeanFactoryPostProcessor"
]
categories = [
    "Spring"
]
image = "bg.jpeg"
draft=false
+++

本篇文章主要是介绍BeanFactoryPostProcessor的实现ConfigurationClassPostProcessor的源码解析，Spring中配置类的解析就是由ConfigurationClassPostProcessor负责完成的，所以这个组件也是Spring中所有类型Bean定义解析的入口，分析其源码可以了解Spring中的bean是如何定义组织的以及最终是如何被程序使用。
<!--more-->

## ConfigurationClassPostProcessor是什么？

![ConfigurationClassPostProcessor](ccpp.png)

 &ensp;&ensp;&ensp;&ensp;上图中的`ConfigurationClassPostProcessor`主要是围绕`ConfigurationClassParser`和`ConfigurationClassBeanDefinitionReader`两个组件构成的。由这两个组件的名称也可以看出整个解析Spring中的bean分为两个大阶段。简单来说，首先是由ConfigurationClassParser解析器从入口配置类开始解析，将最终解析得到的产物暂存在ConfigurationClass中，然后第二个阶段由ConfigurationClassBeanDefinitionReader读取器读取这些产物并注册到bean定义工厂中。

 &ensp;&ensp;&ensp;&ensp;那么在这个过程中，什么是入口配置类呢，或者说哪些类可以作为入口配置类呢？我们又有哪些方式配置Bean呢？解析配置类得到的产物都是些什么呢？什么是ConfigurationClass呢，解析得到的产物又是以什么样的形式存储在ConfigurationClass中的呢？弄清楚了这些问题，第二阶段的读取器无非就是将这些解析产物或者准确的说应应该是bean定义的元数据信息转化成BeanDefinition，调用registry进行注册。

 ## 配置Bean的几种方式
 &ensp;&ensp;&ensp;&ensp;要弄清楚Spring是如何解析注册Bean定义的，首先需要清楚在Spring中，我们都有哪些方式可以进行Bean的配置。一般需要配置Bean的话，是需要有相应的配置类的，所以通过总结配置Bean的方式，我们也能知道配置类有哪些方式可以定义，进而也就清楚了哪些类可以作为入口配类。
 ### @ComponentScan
&ensp;&ensp;&ensp;&ensp;第一种方式使用@ComponentScan注解应该是大家学习Spring时最早接触的声明Bean的方式了。这种方式在指定的包路径下写一个一个的class，在指定的类上使用@Controller、@Service、@Component声明这是bean，最后在启动类上添加@ComponentScan指明要扫描的包路径地址就可以把这些Bean注册到Spring容器中了。
```java
@ComponentScan("com.spring")
public class AppConfig {

}

@Service
public class OrderService {

}

@Component
public class UserService {
    @Autowired
    private OrderService orderService;

    public void test(){
        System.out.println(orderService);
    }
}

public static void main(String[] args) {

    // 创建一个Spring容器
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
    // 获取指定名称的bean
    UserService userService = (UserService) applicationContext.getBean("userService");
    userService.test();
}
```
&ensp;&ensp;&ensp;&ensp;在创建容器上下文时通过其构造函数传入参数AppConfig.class，让容器指定扫描com.spring路径下的所有声明的Bean进行注册。此处OrderService、UserService会被注册到bean工厂后实例化，最终我们可以通过容器上下文AnnotationConfigApplicationContext获取相应实例。

&ensp;&ensp;&ensp;&ensp;在这种方式中，入口配置类就是AppConfig。通过在指定的包路径下声明Bean，最后就会被注册了。这种方式简单明了，也是大家在日常业务开发中使用最多的方式了。

### @Bean
&ensp;&ensp;&ensp;&ensp;通过在指定的方法上使用@Bean注解声明返回的对象需要注册到容器中。@Bean通常跟@ConfigConfiguration一起使用，@ConfigConfiguration声明配置类，@Bean声明bean方法。还有一种非常规的使用方式，@Bean跟@Component一起使用，由于@ConfigConfiguration注解是个复合注解，底层由@Component复合而来，所以本质上@ConfigConfiguration和@Component是一样的。
```java
@Configuration
public class MyConfig {
    @Bean
    public OrderService orderServiceAlias() {
        return new OrderService();
    }
}

public static void main(String[] args) {
    // 创建一个Spring容器
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(MyConfig.class);
    // 获取OrderService实例
    OrderService orderService = (OrderService) applicationContext.getBean("orderServiceAlias");
    orderService.say();
}
```

&ensp;&ensp;&ensp;&ensp;MyConfig类使用@Configuration注解声明为一个配置类，同时在MyConfig中声明了orderServiceAlias方法为bean方法，这样Spring会在容器中注册一个name为orderServiceAlias，类型为OrderService的bean。使用MyConfig作为配置类传入容器AnnotationConfigApplicationContext构造函数。
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {

    @AliasFor(annotation = Component.class)
    String value() default "";


    boolean proxyBeanMethods() default true;
}

```

&ensp;&ensp;&ensp;&ensp;事实上@Configuration注解是个复合注解，它由@Component复合而来，所以此处修改MyConfig类的@Configuration注解为@Component也能达到同样的效果。但是这种方式不如使用@Configuration语义清晰，一般也不推荐这么做。

### @ImportResource
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xmlns:context="http://www.springframework.org/schema/context"
	   xmlns:aop="http://www.springframework.org/schema/aop"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
	   https://www.springframework.org/schema/beans/spring-beans.xsd
	   http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/aop https://www.springframework.org/schema/aop/spring-aop.xsd"
	   >

	<context:component-scan base-package="com.spring"/>

	<bean id="orderService" class="com.zhouyu.service.OrderService"></bean>

</beans>
```

```java
@ImportResource("spring.xml")
public class AppConfig {

}

public static void main(String[] args) {

    // 创建一个Spring容器
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);

    OrderService orderService = (OrderService) applicationContext.getBean("orderService");
    orderService.say();
}
```
&ensp;&ensp;&ensp;&ensp;在AppConfig上使用注解@ImportResource导入一个或多个xml文件中的bean配置，同时将AppConfig作为入口组件配置类传入容器上下文。
### @Import
&ensp;&ensp;&ensp;&ensp;使用@Import可以导入一个或多个bean组件，根据被导入的组件类型又可以细分为如下几种用法。
#### Configuration
&ensp;&ensp;&ensp;&ensp;如果被导入的类是一个普通类，Spring就会把它当作是配置类来处理。那什么是普通类呢？准确的说被导入的类没有实现ImportSelector接口也没有实现ImportBeanDefinitionRegistrar接口，那么Spring就会把这个被导入的类统一当作是配置类来处理，不管这个导入类实际是普通的类还是一个真正的配置类。具体看下面的示例：
```java
public class EmployeeService {
    public void say() {
        System.out.println("hello,i am EmployeeService");
    }
}

public class OrderService {
    public void say() {
        System.out.println("hello,i am OrderService");
    }
}

@Configuration
public class MyConfig {
    @Bean
    public OrderService orderService() {
        return new OrderService();
    }
}

@Import({MyConfig.class, EmployeeService.class})
public class AppConfig {
}

public static void main(String[] args) {
    // 创建一个Spring容器
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
    // 从容器获取OrderService实例
    OrderService orderService = (OrderService) applicationContext.getBean("orderService");
    orderService.say();
    // 从容器获取EmployeeService实例
    // EmployeeService在容器中的beanName为全限定名
    EmployeeService employeeService =(EmployeeService) applicationContext.getBean("com.spring.service.EmployeeService");
    employeeService.say();
}
```
&ensp;&ensp;&ensp;&ensp;EmployeeService和OrderService是两个普通的java类，MyConfig是一个使用了@Configuration声明的配置类，并且在里面声明了Bean方法需要将OrderService注册到容器内。在AppConfig类上使用@Import注解导入MyConfig和EmployeeService，最终在main方法里都能获取OrderService和EmployeeService的实例对象，注意@Bean方法声明的Bean名称为类的全路径。

&ensp;&ensp;&ensp;&ensp;在这种配置场景下，EmployeeService就是一个普通的类，没有继承或实现任何接口和父类，而MyConfig由于使用了@Configuration注解，被我们声明成了一个配置类，目的是要在配置类中进行一些bean的设置。入口的配置类就是AppConfig，当Spring解析到AppConfig上有@Import注解时，会将MyConfig和EmployeeService包装成ConfigurationClass暂存起来，MyConfig对应的ConfigurationClass里面还会存储解析得到的@Bean方法的元数据`MethodMetadata`信息，用于后续注册@Bean声明的方法返回对象。MyConfig类在整个环节中的作用更像是一个中间过渡载体，实际我们最终希望注册的是MyConfig中的@Bean方法声明的返回对象，但是Spring也会注册MyConfig类。对于导入的类，不管是普通的类还是带有@Configuration注解的类Spring在底层都是当作配置类来统一处理，可能你会觉得当解析到EmployeeService时为何不直接把它注册到bean定义工厂中去，还要把它当作一个配置类进一步地解析呢？如果EmployeeService中有@Bean配置的方法呢，那么EmployeeService此时是不是变成了一个配置类呢，虽然没有使用@Configuration显示声明，但是实际上EmployeeService就是一个配置类。
#### ImportSelector

##### DeferredImportSelector
##### ImportBeanDefinitionRegistrar

#### ImportBeanDefinitionRegistrar
