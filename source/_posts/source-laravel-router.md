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

通过这个路由 客户端通过`get`的`http`请求方法 请求`'/user'`这样的URI时  `laravel`会将请求重定向到`User`控制器的`index`方法  最后由该方法返回结果给客户端

首先我们分析下Route这个类 是通过laravel的门面来实现 通过一种简单的方式来绑定访问到容器里的服务router。 其实这里就可以理解为通过Route::get就可以访问到router这个服务的方法 所以说上面的路由等价于:
```php
app()->make('router')->get('user','UserController@index');
```
其中router这个服务是在实例化应用程序时在构造方法里通过注册RoutingServiceProvider绑定到服务容器的

服务的初始化在bootstrap/app/php中时 初始化了一个Application类
```php
$app = new Illuminate\Foundation\Application(
    realpath(__DIR__.'/../')
);
```

在这个Application的初始化函数中我们可以看到服务的注册
```php
public function __construct($basePath = null)
{
    if ($basePath) {
        $this->setBasePath($basePath);
    }

    $this->registerBaseBindings();

    $this->registerBaseServiceProviders();

    $this->registerCoreContainerAliases();
}
```
在注册基础服务提供者时也就是`registerBaseServiceProviders`我们看到这个方法里注册一些相关服务
```php
//注册基础的服务提供器
protected function registerBaseServiceProviders()
{
    $this->register(new EventServiceProvider($this));

    $this->register(new LogServiceProvider($this));

    $this->register(new RoutingServiceProvider($this));
}
```
其中我们就可以看到注册了`RoutingServiceProvider` 那么在这个类中通过`registerRouter`绑定到服务容器的
```php
protected function registerRouter()
{
    $this->app->singleton('router', function ($app) {
        return new Router($app['events'], $app);
    });
}
```
所以说Route的最终的方法斗都是现在\Illuminate\Routing\Router这个类里 在这里类里可以看到一些路由的注册、寻址、调度的方法。

因为路由整个实现的过程无非就是围绕着注册、寻址、调度这样的流程  所以现在可以根据这个过程看下其中的具体实现

### 路由加载
注册路由前需要先加载路由文件 而这里的文件加载是在 App\Providers\RouteServiceProvider 这个服务提供者的boot方法去加载的

```php
class RouteServiceProvider extends ServiceProvider
{
    public function boot()
    {
        parent::boot();
    }

    public function map()
    {
        $this->mapApiRoutes();

        $this->mapWebRoutes();
    }

    protected function mapWebRoutes()
    {
        Route::middleware('web')
             ->namespace($this->namespace)
             ->group(base_path('routes/web.php'));
    }

    protected function mapApiRoutes()
    {
        Route::prefix('api')
             ->middleware('api')
             ->namespace($this->namespace)
             ->group(base_path('routes/api.php'));
    }
}
```

```php
namespace Illuminate\Foundation\Support\Providers;

class RouteServiceProvider extends ServiceProvider
{

    public function boot()
    {
        $this->setRootControllerNamespace();

        if ($this->app->routesAreCached()) {
            $this->loadCachedRoutes();
        } else {
            $this->loadRoutes();

            $this->app->booted(function () {
                $this->app['router']->getRoutes()->refreshNameLookups();
                $this->app['router']->getRoutes()->refreshActionLookups();
            });
        }
    }

    protected function loadCachedRoutes()
    {
        $this->app->booted(function () {
            require $this->app->getCachedRoutesPath();
        });
    }

    protected function loadRoutes()
    {
        if (method_exists($this, 'map')) {
            $this->app->call([$this, 'map']);
        }
    }
}

class Application extends Container implements ApplicationContract, HttpKernelInterface
{
    public function routesAreCached()
    {
        return $this['files']->exists($this->getCachedRoutesPath());
    }

    public function getCachedRoutesPath()
    {
        return $this->bootstrapPath().'/cache/routes.php';
    }
}
```

和很多框架的加载方式一样  laravel先去寻找路由的缓存文件，没有缓存文件再去加载路由。其中缓存文件一般存在`bootstrap/cache/routes.php`

另外我们知道artisan有个commond就是`php artisan route:cache`和`php artisan route:clear`就是针对路由缓存文件的

> 如果路由是闭包方法是不能进行路由缓存的  可以改为控制路由和资源路由

可以看到boot方法里 通过loadRoutes会通过魔术方法调用map方法来加载文件里的路由，map方法在`App\Providers\RouteServiceProvider`类中定义的  而这个类就是继承自`Illuminate\Foundation\Support\Providers\RouteServiceProvider` 通过map方法laravel将路由分成两组  分别对应web服务的路由  和做后端api服务的路由  这两个路由文件的位置就是在项目的routes目录下的web.php  api.php

在5.5版本之前 可以查看5.2版本 可以看到路由文件其实是存放在app/Http/routes.php   这样的改动无非更方便了我们去管理我们的路由文件

既然加载了路由文件 那儿么接下来就是开始了路由的注册

### 路由注册
我们通常且统一的方式是通过Route这个facade调用其中的静态方法去实现各种http请求方法的处理  当然也可以使用上面提到过的那种方法 那么这样的调用实际调用的就是 Illuminate\Routing\Router 里的方法  以为Route这个facade这个是绑定到这个类的  之前也讲过

我们举例几个实现方法来说就是
```php
public function get($uri, $action = null)
{
    return $this->addRoute(['GET', 'HEAD'], $uri, $action);
}

public function post($uri, $action = null)
{
    return $this->addRoute('POST', $uri, $action);
}
```

