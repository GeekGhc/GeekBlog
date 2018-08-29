---
title: Laravel5.5新特性之邮件模板浏览器查看
date: 2017-07-27
categories:
  - Laravel
tags:
    - laravel
---
在Laravel5.5中支持查看我们的HTML邮件模板

`Laravel`之父`Taylor`今天再推特上发布了`Laravel5.5`发布时间改为**8**月份 其实我想说 不发布`Laravel5.5`我就不写代码啦:smile:

不过话说回来 我们还是谈谈带给我们的新特性 其中就是我们可以在浏览器中呈现我们的邮件模板

例如我们去创建一个用户欢迎的邮件模板 在终端执行:

```shell
$ php artisan make:mail UserWelcome --markdown=emails.user.welcome
```

这样我们就在`resources.views.emails.user`目录下就可以生成邮件模板类`UserWelcome`

为了查看浏览器下的浏览效果我们可以在路由中这样去定义
```php?start_inline=1
Route::get('/demo', function () {
    return new App\Mail\UserWelcome();
});
```
下面就是浏览器显示的邮件模板视图
![1](/images/articles/2017-07-25/1.png)

这个新特性将会在`Laravel5.5`中推出  值得期待

想要了解更多的Laravel资讯  可以关注[Laravel News](https://laravel-news.com/render-mailables)
