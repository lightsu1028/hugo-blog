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

主要是介绍spring framwork中的BeanDefinition模块的源码解析。
<!--more-->

## BeanDefinition
`BeanDefinition`表示Bean定义，`BeanDefinition`中存在很多属性用来描述一个Bean的特点。比如：
* class，表示Bean类型
* scope，表示Bean作用域，单例或原型等
* lazyInit：表示Bean是否是懒加载
* initMethodName：表示Bean初始化时要执行的方法
* destroyMethodName：表示Bean销毁时要执行的方法
* ...