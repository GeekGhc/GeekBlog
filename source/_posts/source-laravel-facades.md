---
title: 源码分析之-Facades
date: 2018-12-16
categories:
  - Laravel
tags:
    - source
    - laravel
---

首先看一下`Laravel`官方文档对`Facades`的解释：

> Facades 为应用程序的 服务容器 中可用的类提供了一个「静态」接口。Laravel 本身附带许多的 
>
> facades，甚至你可能在不知情的状况下已经在使用他们！Laravel 「facades」作为在服务容器内基
>
> 类的「静态代理」拥有简洁、易表达的语法优点同时维持着比传统静态方法更高的可测试性和灵活 性。

`Facades`就是一组静态接口或者代理  他们多代表的是一组服务的访问。通过`Facades`可以访问绑定到服务容器里的各种服务。  

之前有谈过路由这个`Facades`，他就是`\Illuminate\Support\Facades\Route`类的别名。他代理的就是注册到服务容器的`router`服务。通过`Router`我们可以访问使用`router`中的各种服务。 我们只需要关注使用，而其中的解析过程则是由laravel内部解析的。这样我们的代码可读性也会高了不少。

我们现在就来看看一个`Facade`注册之后是怎么使用在应用程序里的。当然这之前我们需要关注一个引用启动时`ServiceProvider`这里面的作用

```php
//Class: \Illuminate\Foundation\Http\Kernel
 
protected function sendRequestThroughRouter($request)
{
	$this->app->instance('request', $request);

    Facade::clearResolvedInstance('request');

	  $this->bootstrap();

    //请求最终都会通过Pipeline 进行dispatch
    return (new Pipeline($this->app))
                	->send($request)
                    ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
	                ->then($this->dispatchToRouter());
}

//引导启动Laravel应用程序
public function bootstrap()
{
	if (! $this->app->hasBeenBootstrapped()) {
   	 /**依次执行$bootstrappers中每一个bootstrapper的bootstrap()函数
      $this->bootstrappers = [
            'Illuminate\Foundation\Bootstrap\DetectEnvironment',
            'Illuminate\Foundation\Bootstrap\LoadConfiguration',
            'Illuminate\Foundation\Bootstrap\ConfigureLogging',
            'Illuminate\Foundation\Bootstrap\HandleExceptions',
            'Illuminate\Foundation\Bootstrap\RegisterFacades',
            'Illuminate\Foundation\Bootstrap\RegisterProviders',
            'Illuminate\Foundation\Bootstrap\BootProviders',
      ];*/
      $this->app->bootstrapWith($this->bootstrappers());
    }
}
```

其中会执行 `Illuminate\Foundation\Bootstrap\RegisterFacades` 在这个阶段会依次注册程序中需要用到的`Facades`
```php
// namespace Illuminate\Foundation\Bootstrap;

class RegisterFacades
{
    public function bootstrap(Application $app)
    {
        Facade::clearResolvedInstances();

        Facade::setFacadeApplication($app);

        AliasLoader::getInstance(array_merge(
            $app->make('config')->get('app.aliases', []),
            $app->make(PackageManifest::class)->aliases()
        ))->register();
    }
}
```

这里会通过`AliasLoader`事例 为`Facades`注册别名  而这个别名对应的关系是在定义的`app/config.php`的`配置文件中的 `$aliases`数组中
```php
    'aliases' => [
        'App' => Illuminate\Support\Facades\App::class,
        'Artisan' => Illuminate\Support\Facades\Artisan::class,
        'Auth' => Illuminate\Support\Facades\Auth::class,
        'Blade' => Illuminate\Support\Facades\Blade::class,
        'Broadcast' => Illuminate\Support\Facades\Broadcast::class,
        'Bus' => Illuminate\Support\Facades\Bus::class,
        'Cache' => Illuminate\Support\Facades\Cache::class,
        'Config' => Illuminate\Support\Facades\Config::class,
        ......
    ]
```
进入`AliasLoader`  看一下是如何进行注册这些`Facades`的
```php
// class Illuminate\Foundation\AliasLoader

//获取应用实例
public static function getInstance(array $aliases = [])
{
    if (is_null(static::$instance)) {
        return static::$instance = new static($aliases);
    }

    $aliases = array_merge(static::$instance->getAliases(), $aliases);

    static::$instance->setAliases($aliases);

    return static::$instance;
}

public function register()
{
    if (! $this->registered) {
        $this->prependToLoaderStack();

        $this->registered = true;
    }
}

protected function prependToLoaderStack()
{
    // 把AliasLoader::load()放入自动加载函数队列中，并置于队列头部
    spl_autoload_register([$this, 'load'], true, true);
}
```

`spl_autoload_register` 函数是实现自动加载未定义类功能的的重要方法 这个函数的参数如下

#### autoload_function
这是一个函数【方法】名称，可以是字符串或者数组（调用类方法使用）。这个函数（方法）的功能就是，来把需要`new` 的类文件包含`include(requeire)`进来，这样`new`的时候就不会找不到文件了。其实就是封装整个项目的include和require功能。

#### throw
此参数设置了 `autoload_function` 无法成功注册时， `spl_autoload_register()`是否抛出异常。

#### prepend
如果是 `true`，`spl_autoload_register()` 会添加函数到队列之首，而不是队列尾部。

上面的代码就是 `AliasLoader` 将load方法注册到`SPL __autoload`函数队列的头部。

这样的话当我们需要一个服务类的时候会调用`load`方法引入进来 看一下`load`方法的源码
```php
public function load($alias)
{
   if (static::$facadeNamespace && strpos($alias, static::$facadeNamespace) === 0) {
            $this->loadFacade($alias);
            return true;
        }
    if (isset($this->aliases[$alias])) {
        return class_alias($this->aliases[$alias], $alias);
    }
}
```
load方法里把$aliases里的配置Facade类创建了对应的别名，比如说我们使用Auth类的时候 laravel内部会通过`AliasLoader` 的`load`方法为 `Illuminate\Support\Facades\Auth` 这个类创建一个别名类`Auth`。

所以我们在应用程序中使用的Auth类其实就是使用的 `Illuminate\Support\Facades\Auth` 这个类

### 解析Facade代理服务
之前所提到的 我们已经成功将`Facade`服务注册了  那么我们再使用一些`Facades` 比如`Route::get('/',function(){})`这样的方法时 那么此时如何通过Route这个静态代理解析到里面的服务呢。这个就是下面要说的`laravel`里的隐式解析。

我们说了Route其实是对应的`Illuminate\Support\Facades\Route`这个类 可以看下里面都有什么
```php
namespace Illuminate\Support\Facades;
class Route extends Facade
{
    protected static function getFacadeAccessor()
    {
        return 'router';
    }
}
```
我们主要看下`getFacadeAccessor`这个方法  你会发现无论是`Route`还是`Auth`等这些`Facade`所对应的类都会有这个方法

这里面并没有我们所预期的get、post、patch这些方法  这个我们就去他的父类 `Illuminate\Support\Facades\Facade`看看  依然没有  我们知道`PHP`如果执行没有给定的方法  回去执行他的魔术方法  也就是 `__callStatic`静态方法
```php
namespace Illuminate\Support\Facades;

abstract class Facade
{
    public static function __callStatic($method, $args)
    {
        $instance = static::getFacadeRoot();

        if (! $instance) {
            throw new RuntimeException('A facade root has not been set.');
        }

        return $instance->$method(...$args);
    }
    
    //获取Facade根对象
    public static function getFacadeRoot()
    {
        return static::resolveFacadeInstance(static::getFacadeAccessor());
    }
    
    /**
     * 从服务容器里解析出Facade对应的服务
     */
    protected static function resolveFacadeInstance($name)
    {
        if (is_object($name)) {
            return $name;
        }

        if (isset(static::$resolvedInstance[$name])) {
            return static::$resolvedInstance[$name];
        }

        return static::$resolvedInstance[$name] = static::$app[$name];
    }
}
```

我们可以发现 `Illuminate\Support\Facades\Facade` 这个父类是一个抽象类  这样的话我们可以根据自己的需要去增加新的子系统外观类 并让外观类可以正确代理其对应的子系统。

比如这里的`Route`那么最终的`instance`就是 调用的 `resolveFacadeInstance('router')`  也就是子类`Route Facade`里设置的`accessor`(字符串`router`)  然后从服务容器中解析到对应的服务。

而`router`服务是在应用程序初始化时的`registerBaseServiceProviders`阶段被`\Illuminate\Routing\RoutingServiceProvider`注册到服务容器里的: 具体可以参考`laravel` 路由的源码分析。
```php
class RoutingServiceProvider extends ServiceProvider
{
    /**
     * Register the service provider.
     *
     * @return void
     */
    public function register()
    {
        $this->registerRouter();
		......
    }

    /**
     * Register the router instance.
     *
     * @return void
     */
    protected function registerRouter()
    {
        $this->app->singleton('router', function ($app) {
            return new Router($app['events'], $app);
        });
    }
    ......
}
```
`router`服务对应的类就是`\Illuminate\Routing\Router`, 所以`Route Facade`实际上代理的就是这个类

所以通过`Route Facade`访问的`get`、`post`方法都是访问的`\Illuminate\Routing\Router`这个类的方法

值的注意的是

- 1.解析服务时用的`static::$app`是在最开始的`RegisterFacades`里设置的，它引用的是服务容器。

- 2.`static::$app['router']`  以数组访问的形式能够从服务容器解析出`router`服务是因为服务容器实现了`SPL`的`ArrayAccess`接口, 对这个没有概念的可以看下官方文档[ArrayAccess](https://www.php.net/manual/zh/class.arrayaccess.php)

对于`ArrayAccess` 需要实现其四个方法 `offsetExists`、`offsetGet`、`offsetSet`、`offsetUnset`

通过重写这四个方法 那么我们就可以对服务对象进行存取  和操作数组一样  只不过我们操作的事的各种子服务而已


