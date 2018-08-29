---
title: Laravel5.5新特性之自定义验证规则
date: 2017-07-24
categories:
  - Laravel
tags:
    - laravel
---
在Laravel5.5中支持了自定义验证规则以此来作为Validator:extend进行验证规则的替换方法

## 简介
`Laravel5.5`正式版这些天也即将发布 他给我们带来的新特性也十分一批开发者激动不已 我们先来谈谈所支持的自定义验证规则

在`Laravel5.5`中新增了`php artisan make:rule`这个命令  这个命令可以很方便的生成我们自己定义的验证规则

在自己的实现的验证规则里我们可以自己去定义业务判断逻辑

## 着手实现
1.创建项目
因为`Laravel5.5`还没有正式发布 不过我们是可以拿到它的源码 在命令行执行
```shell
$ laravel new laravel5-5 --dev
```

过会儿便可以下载完成 下载完成之后我们会看见有个报错页面 嗯哼
![1](/images/articles/2017-07-24/1.png)

这个报错看起来很酷有木有  我们可以准备看到报错信息和文件位置  当然还可以直接`google`到相应的解决方案 `nice~`

当然这里的报错是因为我们还没有生成`app:key`   `so`在项目根目录下执行
```shell
$ php artisan key:generate
```
重新启动一下我们的项目就**ok**啦

2.创建规则
作为事例  我们可以去创建一个简单的字符串长度判断的验证 在终端执行:
```shell
$ php artisan make:rule LengthRule
```
这个时候我们会发现在app目录下的rule文件夹下生成了LengthRule.php的验证文件

而我们主要的验证方法就是在pass方法中去实现
```php?start_inline=1
public function passes($attribute, $value)
{
    return $value>6;
}
```
当然在下面的`message`方法中去定义我们的验证结果的消息
```php?start_inline=1
public function message()
{
    return '字符串长度不能低于6位';
}
```
这样的话我们就可以在控制器的方法中去使用这个验证规则
```php?start_inline=1
public function testRule()
{
    $this->validate(request(),[
        'name'=>[new LengthRule()]
    ]);
}
```

如果想要测试简单的方法就是在路由中新建一个方法 对于参数可以使用借助`postman`等工具提交参数请求

值得注意的是如果这个参数字段为空 那么这个验证规则是不会执行的 要想不存在这个字段也去执行这个这个验证规则

那么我们需要去实现`ImplicitRule`契约 比方说
```php?start_inline=1
class LengthRule implements ImplicitRule
{
    public function __construct()
    {
        //
    }
    public function passes($attribute, $value)
    {
        return $value>6;
    }
    public function message()
    {
        return '字符串长度不能低于6位';
    }
}
```

> 对于想了解这样的新特性最好的办法就是去`Github`上的这个项目的[PR](https://github.com/laravel/framework/pull/19155/files)

