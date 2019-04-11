---
title: 源码分析之-Facades
date: 2018-12-16
categories:
  - Laravel
tags:
    - source
    - laravel
---

首先看一下Laravel官方文档对Facades的解释：

> Facades 为应用程序的 服务容器 中可用的类提供了一个「静态」接口。Laravel 本身附带许多的 
>
> facades，甚至你可能在不知情的状况下已经在使用他们！Laravel 「facades」作为在服务容器内基
>
> 类的「静态代理」拥有简洁、易表达的语法优点同时维持着比传统静态方法更高的可测试性和灵活 性。

`Facades`就是一组静态接口或者代理  他们多代表的是一组服务的访问。通过`Facades`可以访问绑定到服务容器里的各种服务。  

之前有谈过路由这个`Facades`，他就是`\Illuminate\Support\Facades\Route`类的别名。他代理的就是注册到服务容器的`router`服务。通过`Router`我们可以访问使用`router`中的各种服务。 我们只需要关注使用，而其中的解析过程则是由laravel内部解析的。这样我们的代码可读性也会高了不少。

我们现在就来看看一个`Facade`注册之后是怎么使用在应用程序里的。当然这之前我们需要关注一个引用启动时`ServiceProvider`这里面的作用

