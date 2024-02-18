+++
title = "ClassPathBeanDefinitionScanner主流程源码解析"
date = "2023-01-25"
description = "Spring BeanDefinition模块扫描器源码解析"
tags = [
    "Spring",
    "BeanDefinition"
]
categories = [
    "Spring"
]
image = "bd.jpeg"
draft=true
+++

本篇文章主要是介绍Spring在生成BeanDefinition的过程中，涉及到的核心组件ClassPathBeanDefinitionScanner的主流程源码解析。ClassPathBeanDefinitionScanner涉及到的相关底层API使用我在另一篇博文中已经介绍过了[Spring源码BeanDefinition解析之ClassPathBeanDefinitionScanner]({{< relref "springsrc-beanDefinition-ClassPathBeanDefinitionScanner#前言" >}})
<!--more-->

## 扫描解析入口方法
Spring上下文容器在扫描时会调用`org.springframework.context.annotation.ClassPathBeanDefinitionScanner#scan`进行解析指定包路径下面的候选
定义，具体Spring是在哪调用该方法进入扫描解析流程的后续再详细分析，本篇专注解析逻辑本身。
```java{.line-numbers}
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
scan方法中并没有核心的解析逻辑，主要做了两件事，第一件事调用doScan(basePackages)方法解析用户包路径下的自定义的BeanDefinition。第二件事就是向上下文容器中直接注册注解配置相关的处理器BeanDefinition，后续就可以直接使用这些infra组件辅助完成容器的后续初始化工作。
## 注册基础组件
Spring默认会进行基础组件BeanDefinition的手动注册，这些基础组件的BeanDefinition后续会直接实例化辅助Spring容器初始化，具体的方法逻辑如下：
```java{.line-numbers}
org.springframework.context.annotation.AnnotationConfigUtils#registerAnnotationConfigProcessors(org.springframework.beans.factory.support.BeanDefinitionRegistry, java.lang.Object)

public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {

    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
    if (beanFactory != null) {

        // 设置beanFactory的OrderComparator为AnnotationAwareOrderComparator
        // 它是一个Comparator，是一个比较器，可以用来进行排序，比如new ArrayList<>().sort(Comparator);
        if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
            beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
        }
        // 用来判断某个Bean能不能用来进行依赖注入
        if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
            beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        }
    }

    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

    // 注册ConfigurationClassPostProcessor类型的BeanDefinition
    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 注册AutowiredAnnotationBeanPostProcessor类型的BeanDefinition
    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 注册CommonAnnotationBeanPostProcessor类型的BeanDefinition
    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 注册PersistenceAnnotationBeanPostProcessor类型的BeanDefinition
    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
    if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition();
        try {
            def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
                    AnnotationConfigUtils.class.getClassLoader()));
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
        }
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 注册EventListenerMethodProcessor类型的BeanDefinition，用来处理@EventListener注解的
    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
    }

    // 注册DefaultEventListenerFactory类型的BeanDefinition，用来处理@EventListener注解的
    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
    }

    return beanDefs;
}
```
* 第7-18行：向容器上下文中注入AnnotationAwareOrderComparator和ContextAnnotationAutowireCandidateResolver两个组件，都是跟依赖注入功能相关，第一个组件在上篇文章介绍扫描器时已经详细介绍了，这里就不赘述了，第二个组件会在解析依赖注入功能时再详细介绍。
* 第23-72行：向容器中手动设置各种类型的postProcessor后置扩展点处理器作为infra组件：
    * `org.springframework.beans.factory.config.BeanFactoryPostProcessor`：ConfigurationClassPostProcessor、EventListenerMethodProcessor
    * `org.springframework.beans.factory.config.BeanPostProcessor`：AutowiredAnnotationBeanPostProcessor、CommonAnnotationBeanPostProcessor、PersistenceAnnotationBeanPostProcessor
    * `org.springframework.context.event.EventListenerFactory`:DefaultEventListenerFactory。

这些基础的内置infra扩展点处理器后续会在容器初始化的不同生命周期阶段发挥作用，到时用到再进行详细分析。Spring在扫描用户自定义的bean前会预先调用registerPostProcessor方法进行手动注册。具体的注册逻辑又是怎样的呢？

```java
org.springframework.context.annotation.AnnotationConfigUtils#registerPostProcessor

private static BeanDefinitionHolder registerPostProcessor(
			BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

    definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registry.registerBeanDefinition(beanName, definition);  // BeanDefinitinoMap
    return new BeanDefinitionHolder(definition, beanName);
}
```
这里我们先简单的看一下infra的beanDefinition注册步骤，详细的注册逻辑在后面注册用户自定义的bean时再分析。首先这些infra的组件beanDefinition会被创建成`org.springframework.beans.factory.support.RootBeanDefinition`，RootBeanDefinition是可以直接被容器拿来进行IOC的beanDefinition类型，无须再进行其他的解析处理，相当于是已经解析后的成品beanDefinition，其他类型的beanDefinition在IOC时最后都会被转换（进一步解析）成RootBeanDefinition，这些不同类型的beanDefinition我会在新的文章中详细介绍。

beanDefinition的类型会被设置ROLE_INFRASTRUCTURE表示当前bean的类型是基础组件。
* ROLE_APPLICATION：表示bean是应用组成级别的，通常用户自定义的bean会被设置成此角色
* ROLE_SUPPORT：表示bean是支持配置级别的
* ROLE_INFRASTRUCTURE：通常表示内部功能级别的bean，这些bean不会被用户使用

真正进行注册beanDefinition则是调用容器的registerBeanDefinition将definition注册到容器的BeanDefinitinoMap中去。

## 注册用户自定义beanDefinition
```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {

        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);

        for (BeanDefinition candidate : candidates) {
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());

            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);

            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition) {
                // 解析@Lazy、@Primary、@DependsOn、@Role、@Description
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }

            // 检查Spring容器中是否已经存在该beanName
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);

                // 注册
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
```
* 第4-6行：扫描basePackage下的资源文件生成BeanDefinition，findCandidateComponents找出候选的BeanDefinition是核心的扫描解析逻辑，下面详细分析。
* 第9-10行：解析bean的scope属性
* 第12行：调用BeanNameGenerator给bean生成名字

## 解析资源文件生成BeanDefinition
```java
org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#findCandidateComponents

public Set<BeanDefinition> findCandidateComponents(String basePackage) {
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {
        return addCandidateComponentsFromIndex(this.componentsIndex, basePackage);
    }
    else {
        return scanCandidateComponents(basePackage);
    }
}
```
* 第4-6行：使用索引扫描BeanDefinition
* 第8行：通过扫描资源文件即用户的class文件生成BeanDefinition

### 索引扫描
TODO
### 扫描候选组件
将资源文件解析成BeanDefinition的功能主要由scanCandidateComponents方法完成。入参basePackage是包路径，返回值为解析的BeanDefinition集合。源码如下：
```java
private Set<BeanDefinition> scanCandidateComponents(String basePackage) {
    Set<BeanDefinition> candidates = new LinkedHashSet<>();
    try {
        // 获取basePackage下所有的文件资源
        String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +
                resolveBasePackage(basePackage) + '/' + this.resourcePattern;

        Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);

        boolean traceEnabled = logger.isTraceEnabled();
        boolean debugEnabled = logger.isDebugEnabled();
        for (Resource resource : resources) {
            if (traceEnabled) {
                logger.trace("Scanning " + resource);
            }
            if (resource.isReadable()) {
                try {
                    MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);
                    // excludeFilters、includeFilters判断
                    if (isCandidateComponent(metadataReader)) { // @Component-->includeFilters判断
                        ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);
                        sbd.setSource(resource);

                        if (isCandidateComponent(sbd)) {
                            if (debugEnabled) {
                                logger.debug("Identified candidate component class: " + resource);
                            }
                            candidates.add(sbd);
                        }
                        else {
                            if (debugEnabled) {
                                logger.debug("Ignored because not a concrete top-level class: " + resource);
                            }
                        }
                    }
                    else {
                        if (traceEnabled) {
                            logger.trace("Ignored because not matching any filter: " + resource);
                        }
                    }
                }
                catch (Throwable ex) {
                    throw new BeanDefinitionStoreException(
                            "Failed to read candidate component class: " + resource, ex);
                }
            }
            else {
                if (traceEnabled) {
                    logger.trace("Ignored because not readable: " + resource);
                }
            }
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException("I/O failure during classpath scanning", ex);
    }
    return candidates;
}
```
* 第5-8行：拼接类路径地址，最终得到classpath*:xxx/**/*.class这样的一个类似通配符的地址，xxx是你传入的包名，可以通过@ComponentScan注解指定。getResourcePatternResolver()会获取ResourcePatternResolver进行多个资源的解析，如果扫描器设置了resourcePatternResolver就直接使用，没有则使用AnnotationConfigApplicationContext最为默认的ResourcePatternResolver进行资源解析。具体的资源解析器相关功能可参照[Spring源码BeanDefinition解析之ClassPathBeanDefinitionScanner]({{< relref "springsrc-beanDefinition-ClassPathBeanDefinitionScanner#ResourceLoader" >}})这篇文章相关章节。

* 第18-20行:从MetadataReaderFactory获取MetadataReader，默认使用CachingMetadataReaderFactory，CachingMetadataReaderFactory会使用`Map<Resource, MetadataReader> metadataReaderCache`作为缓存存储Resource跟MetadataReader，Spring默认使用`org.springframework.core.type.classreading.SimpleMetadataReader`。isCandidateComponent方法使用excludeFilters、includeFilters、conditionEvaluator判断扫描到的资源文件是否匹配，includeFilters默认会匹配含有@Component的资源。过滤器和条件注解的具体使用参考[Spring源码BeanDefinition解析之ClassPathBeanDefinitionScanner]({{< relref "springsrc-beanDefinition-ClassPathBeanDefinitionScanner#ResourceLoader" >}})这篇文章相关章节。

* 第21-28行：将上述步骤匹配的资源封装成`org.springframework.context.annotation.ScannedGenericBeanDefinition`，24行又有一个isCandidateComponent方法，会进一步在判断扫描得到的资源是否符合一个候选组件，符合的话最终加到candidates集合中返回出去。看下这个isCandidateComponent的主要逻辑：

```java
org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider#isCandidateComponent(org.springframework.beans.factory.annotation.AnnotatedBeanDefinition)

protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
    AnnotationMetadata metadata = beanDefinition.getMetadata();
    return (metadata.isIndependent() && (metadata.isConcrete() ||
            (metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));
}
```
这里判断符合候选组件的条件需要同时满足两个条件：
1. 候选组件是独立的类，如果是嵌套的内部类那必须是静态内部类。
2. 候选组件是具体的类，即非接口非抽象类；或者是抽象类的话，方法上必须含有Lookup注解。

scanCandidateComponents方法筛选候选组件时为什么要分两个isCandidateComponent方法判断呢而不是写在一个方法里面呢？第一个判断方法主要是进行外部的exclude和include过滤器的筛选，并不进行组件本身的属性条件筛选，第二个判断方法才针对类的自身属性进行具体的条件判断，诸如类的结构、是否为接口或抽象类等条件。***注意这两个判断方法都使用了protected关键字，是可以被子类重写的，如果你有自己的特殊扫描逻辑，可以通过重写这两个判断方法使用你自己的筛选逻辑***。

## scope属性解析
扫描解析出候选的BeanDefinition后，doScan方法会使用`org.springframework.context.annotation.ScopeMetadataResolver`解析scope相关的属性并设置到BeanDefinition中去。
```java
// 解析scope属性元数据信息
ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
// 设置BeanDefinition是singleton还是prototype
candidate.setScope(scopeMetadata.getScopeName());
```
这里使用ScopeMetadataResolver的实现`org.springframework.context.annotation.AnnotationScopeMetadataResolver`看下scope的元数据解析是如何实现的：
```java
org.springframework.context.annotation.AnnotationScopeMetadataResolver#resolveScopeMetadata

public ScopeMetadata resolveScopeMetadata(BeanDefinition definition) {
    ScopeMetadata metadata = new ScopeMetadata();
    if (definition instanceof AnnotatedBeanDefinition) {
        AnnotatedBeanDefinition annDef = (AnnotatedBeanDefinition) definition;
        AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(
                annDef.getMetadata(), this.scopeAnnotationType);
        if (attributes != null) {
            metadata.setScopeName(attributes.getString("value"));
            ScopedProxyMode proxyMode = attributes.getEnum("proxyMode");
            if (proxyMode == ScopedProxyMode.DEFAULT) {
                proxyMode = this.defaultProxyMode;
            }
            metadata.setScopedProxyMode(proxyMode);
        }
    }
    return metadata;
}
```
* 第6-8行：获取AnnotatedBeanDefinition的元数据信息AnnotationMetadata，通过注解工具类AnnotationConfigUtils获取Scope属性的元数据信息AnnotationAttributes，AnnotationAttributes实际上是一个LinkedHashMap<String, Object>，key就是注解中的属性名称，value就是注解中的属性值。***所以一个AnnotationAttributes实际就是一个注解的元数据信息集合***。

* 第9-16行：有了Scope属性的元数据信息，提取value值和proxyMode值设置到ScopeMetadata中。Scope注解中的proxyMode值是个枚举值，用于指定在创建代理对象时使用的代理模式，它有三种可能的取值：
    * proxyMode = ScopedProxyMode.NO：这表示不要使用代理来包装bean，直接返回原始的bean对象，如果不设置proxyMode属性，默认值就是这个。
    * proxyMode = ScopedProxyMode.INTERFACES：这表示使用JDK动态代理来包装bean，并且只暴露bean实现的接口。
    * proxyMode = ScopedProxyMode.TARGET_CLASS：这表示使用CGLIB来包装bean，并保留类的结构。

## beanName生成规则
在doScan方法中，使用`org.springframework.beans.factory.support.BeanNameGenerator#generateBeanName`生成bean的名称，BeanNameGenerator接口有多个实现，这里使用`org.springframework.context.annotation.AnnotationBeanNameGenerator`作为示例参考：
```java
org.springframework.context.annotation.AnnotationBeanNameGenerator#generateBeanName

public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
    if (definition instanceof AnnotatedBeanDefinition) {
        // 获取注解所指定的beanName
        String beanName = determineBeanNameFromAnnotation((AnnotatedBeanDefinition) definition);
        if (StringUtils.hasText(beanName)) {
            // Explicit bean name found.
            return beanName;
        }
    }
    // Fallback: generate a unique default bean name.
    return buildDefaultBeanName(definition, registry);
}
```
* 第2-9行：如果类使用了注解配置，则使用注解方式生成beanName。
* 第11行：非注解方式定义bean，则使用默认的beanName生成规则。

### 注解方式生成beanName
```java
org.springframework.context.annotation.AnnotationBeanNameGenerator#determineBeanNameFromAnnotation

protected String determineBeanNameFromAnnotation(AnnotatedBeanDefinition annotatedDef) {
    AnnotationMetadata amd = annotatedDef.getMetadata();
    Set<String> types = amd.getAnnotationTypes();
    String beanName = null;
    for (String type : types) {
        AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(amd, type);
        if (attributes != null) {
            Set<String> metaTypes = this.metaAnnotationTypesCache.computeIfAbsent(type, key -> {
                Set<String> result = amd.getMetaAnnotationTypes(key);
                return (result.isEmpty() ? Collections.emptySet() : result);
            });
            if (isStereotypeWithNameValue(type, metaTypes, attributes)) {
                Object value = attributes.get("value");
                if (value instanceof String) {
                    String strVal = (String) value;
                    if (StringUtils.hasLength(strVal)) {
                        if (beanName != null && !strVal.equals(beanName)) {
                            throw new IllegalStateException("Stereotype annotations suggest inconsistent " +
                                    "component names: '" + beanName + "' versus '" + strVal + "'");
                        }
                        beanName = strVal;
                    }
                }
            }
        }
    }
    return beanName;
}
```
* 第2-3行：获取类上的注解元数据信息和所有的注解全限定名称
* 第6-11行：找出这些注解的属性元信息，如果注解是复合注解，比如@Service、@Controller这种那么再找出这些复合注解的元注解信息，isStereotypeWithNameValue根据这些数据判断注解中是否包含Component的信息
```java
protected boolean isStereotypeWithNameValue(String annotationType,
        Set<String> metaAnnotationTypes, @Nullable Map<String, Object> attributes) {

    boolean isStereotype = annotationType.equals(COMPONENT_ANNOTATION_CLASSNAME) ||
            metaAnnotationTypes.contains(COMPONENT_ANNOTATION_CLASSNAME) ||
            annotationType.equals("javax.annotation.ManagedBean") ||
            annotationType.equals("javax.inject.Named");

    return (isStereotype && attributes != null && attributes.containsKey("value"));
}
```
### 默认方式生成beanName

