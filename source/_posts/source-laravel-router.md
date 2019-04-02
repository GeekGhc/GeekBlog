---
title: 源码分析之-Laravel路由
date: 2018-12-10
categories:
  - Laravel
tags:
    - source
    - laravel
---

首先这里以`laravel5.5`版本为例  初始化新的项目

路由是外界访问`laravel`应用程序的通道 通过指定`URI`和`HTTP`请求方法 那么就可以访问项目应用程序的处理方法或者闭包。现在我们可以去研究下在`Laravel`中是如何处理这些请求 并重新解析到对应的方法体。

以一个我们通常的访问形式说起
```php
Route::get('/user', 'UsersController@index');
```

通过这个路由 客户端通过`get`的`http`请求方法 请求`'/user'`这样的URI时  `laravel`会将请求重定向到`User`控制器的`index`方法