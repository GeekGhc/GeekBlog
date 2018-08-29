---
title: Laravel5.5新特性之Blade::if 自定义指令
date: 2017-07-27
categories:
  - Laravel
tags:
    - laravel
---
在Laravel5.5中支持了简化视图if的自定义的指令

`Laravel`在下个月才能发布  不过不影响我们对新特性的期待 再此之前说了两个新的特性 现在再来啊介绍一个新的特性

就是在**5.5**中支持了自定义标签 这样就简化了之前在`View`中的`if`标签

那么问题来了 首先我们应该还是想到的是这个新特性的应用场景是什么 举个栗子来说

当我们需要判断是否是管理员还是普通用户 那么视图的展现会有所区分的

通常来说我们可能在视图中这样去写
```php?start_inline=1
@if(auth()->check() && auth()->user()->isAdmin())
  管理员
@else
  普通用户
@endif
```
其实这样的判断逻辑就是用户是否登录并且该用户是不是管理员身份  如果两者都符合 那么才能判断该用户是可以有管理员权限的

`isAdmin()`这个`function` 可以在`model`定义  用户判断用户到底属于什么身份

其实在**5.5**中我们有更好的解决方案 在 `AppServiceProvider::boot()` 方法中
```php?start_inline=1
use Illuminate\Support\Facades\Blade;

Blade::if('isAdmin', function () {
    return auth()->check() && auth()->user()->isAdmin();
});
```
这样我们就把之前在视图if中的判断逻辑搬到这边来了 那么我们现在在视图中完全可以这样写:
```php?start_inline=1
@isAdmin
    管理员
@else
    普通用户
@endIsAdmin
```
这样我们就完成了自定义的标签  

当然其实还有这样的一个应用场景就是 线上和线下的生产环境 也就是我们通常所说的开发环境和应用环境

这在`Laravel5.5`中更为简化了
```php?start_inline=1
Blade::if('prod', function () {
    return app()->environment('production');
});
```
当然还可以传递参数这样检查更为可控
```php?start_inline=1
Blade::if('env', function ($env) {
    return app()->environment($env);
});
```

相关具体内容请查看 [https://laravel-news.com/bladeif](https://laravel-news.com/bladeif)