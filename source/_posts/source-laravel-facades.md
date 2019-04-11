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