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

首先我们分析下`Route`这个类 是通过`laravel`的门面来实现 通过一种简单的方式来绑定访问到容器里的服务`router`。 其实这里就可以理解为通过`Route::get`就可以访问到`router`这个服务的方法 所以说上面的路由等价于:
```php
app()->make('router')->get('user','UserController@index');
```
其中`router`这个服务是在实例化应用程序时在构造方法里通过注册`RoutingServiceProvider`绑定到服务容器的

服务的初始化在`bootstrap/app/php`中时 初始化了一个`Application`类
```php
$app = new Illuminate\Foundation\Application(
    realpath(__DIR__.'/../')
);
```

在这个`Application`的初始化函数中我们可以看到服务的注册
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
所以说`Route`的最终的方法斗都是现在`\Illuminate\Routing\Router`这个类里 在这里类里可以看到一些路由的注册、寻址、调度的方法。

因为路由整个实现的过程无非就是围绕着注册、寻址、调度这样的流程  所以现在可以根据这个过程看下其中的具体实现

### 路由加载
注册路由前需要先加载路由文件 而这里的文件加载是在 `App\Providers\RouteServiceProvider` 这个服务提供者的`boot`方法去加载的

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

和很多框架的加载方式一样  `laravel`先去寻找路由的缓存文件，没有缓存文件再去加载路由。其中缓存文件一般存在`bootstrap/cache/routes.php`

另外我们知道`artisan`有个`commond`就是`php artisan route:cache`和`php artisan route:clear`就是针对路由缓存文件的

> 如果路由是闭包方法是不能进行路由缓存的  可以改为控制路由和资源路由

可以看到`boot`方法里 通过`loadRoutes`会通过魔术方法调用`map`方法来加载文件里的路由，`map`方法在`App\Providers\RouteServiceProvider`类中定义的  而这个类就是继承自`Illuminate\Foundation\Support\Providers\RouteServiceProvider` 

通过`map`方法`laravel`将路由分成两组  分别对应`web`服务的路由  和做后端`api`服务的路由  这两个路由文件的位置就是在项目的`routes`目录下的`web.php`  `api.php`

在**5.5**版本之前 可以查看**5.2**版本 可以看到路由文件其实是存放在`app/Http/routes.php`   这样的改动无非更方便了我们去管理我们的路由文件

既然加载了路由文件 那儿么接下来就是开始了路由的注册

### 路由注册
我们通常且统一的方式是通过`Route`这个`facade`调用其中的静态方法去实现各种`http`请求方法的处理  当然也可以使用上面提到过的那种方法 那么这样的调用实际调用的就是 `Illuminate\Routing\Router` 里的方法  因为`Route`这个`facade`这个是绑定到这个类的  之前也讲过

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
public function put($uri, $action = null)
{
    return $this->addRoute('PUT', $uri, $action);
}
```
可以看到的是路由的注册都是通过addRoute这个方法去进行注册的
```php
protected function addRoute($methods, $uri, $action)
{
    return $this->routes->add($this->createRoute($methods, $uri, $action));
}
```
而这个`addRoute`方法则是会将路由注册到`RouteCollection` 当然之前会调用路由的创建方法`createRoute`

```php
protected function createRoute($methods, $uri, $action)
{
    if ($this->actionReferencesController($action)) {
        $action = $this->convertToControllerAction($action);
    }

    $route = $this->newRoute(
        $methods, $this->prefix($uri), $action
    );
    if ($this->hasGroupStack()) {
        $this->mergeGroupAttributesIntoRoute($route);
    }

    $this->addWhereClausesToRoute($route);

    return $route;
}

```
而这个创建路由的方法接收三个参数 分别是路由的请求方法 `uri`以及执行体  这执行可以使未解析的资源路径 也可以是一个闭包方法

其中我们最常见的解析到对应的控制器的方法是由`($this->actionReferencesController($action)`进行解析转换的  这里会判断不是一个闭包`(Closure)` 也就是如果是`string`类型的话

值得注意的是  这里的`action` 我们在路由注册时会以数组 字符串以及闭包的形式传递 那么如果是数组也就是比如`['uses' => 'HomeController@index, 'middleware' => 'auth']`这样的形式的 以及字符串类型的也就是比如`HomController@index`这种形式的都会通过`convertToControllerAction`这个方法进行解析成action数组  最后的保存形式就是
```php
[
    'uses' => 'App\Http\Controllers\HomeController@index',
    'controller' => 'App\Http\Controllers\HomeController@index'
]
```
经过转换解析之后  会补充控制器的完整的命名空间 构建完`action`数组之后那么之后就是创建路由了

创建路由会由之前的指定的`http`方法`uri`字符串以及转换后的`action`数组作为参数进行创建 `\Illuminate\Routing\Route`类的实例:
```php
$route = $this->newRoute(
            $methods, $this->prefix($uri), $action
        );

protected function newRoute($methods, $uri, $action)
{
    return (new Route($methods, $uri, $action))
                ->setRouter($this)
                ->setContainer($this->container);
}
```
路由创建完成后再添加到之前所说的`RouteCollection`中去:
```php
protected function addRoute($methods, $uri, $action)
{
    return $this->routes->add($this->createRoute($methods, $uri, $action));
}
```
这里的`$this->routes`就是类在初始化时的`RouteCollection`对象
```php
public function __construct(Dispatcher $events, Container $container = null)
{
    $this->events = $events;
    $this->routes = new RouteCollection;
    $this->container = $container ?: new Container;
}
```
而新添加的路由 也就是一个`Route`对象会更新`RouteCollection`中的`routes`、`allRoutes`、`nameList`和`actionList`属性
```php
class RouteCollection implements Countable, IteratorAggregate
{
    public function add(Route $route)
    {
        $this->addToCollections($route);

        $this->addLookups($route);

        return $route;
    }

    protected function addToCollections($route)
    {
        $domainAndUri = $route->getDomain().$route->uri();

        foreach ($route->methods() as $method) {
            $this->routes[$method][$domainAndUri] = $route;
        }

        $this->allRoutes[$method.$domainAndUri] = $route;
    }

    protected function addLookups($route)
    {
        $action = $route->getAction();

        if (isset($action['as'])) {
            //如果时命名路由，将route对象映射到以路由名为key的数组值中方便查找
            $this->nameList[$action['as']] = $route;
        }

        if (isset($action['controller'])) {
            $this->addToActionList($action, $route);
        }
    }
    protected function addToActionList($action, $route)
    {
        $this->actionList[trim($action['controller'], '\\')] = $route;
    }

}
```

既然更新了`RouteCollection`这四个属性 下面就看下这个属性的作用

routes中存放了HTTP请求方法与路由对象的映射,就像这样:
```php
[
    'GET' => [
        $routeUri1 => $routeObj1
        ...
    ]
    ...
]
```
`allRoutes`属性里存放的内容时将`routes`属性里的二维数组变成一维数组后的内容:
```php
[
    'GET' . $routeUri1 => $routeObj1
    'GET' . $routeUri2 => $routeObj2
    ...
]
```
`nameList`是路由名称与路由对象的一个映射表
```php
[
    $routeName1 => $routeObj1
    ...
]
```
`actionList`是路由控制器方法字符串与路由对象的映射表
```php
[
    'App\Http\Controllers\ControllerOne@ActionOne' => $routeObj1
]
```
这样一来就可以完成了一个路由的注册

### 路由寻址

首先我们可以预先了解一个概念就是 我们知道`laravel`在请求路由到最终的返回之间有一层中间件 `HTTP`请求是在经过`Pipeline`通道上的中间件的前置操作后到达目的地:
```php
//Illuminate\Foundation\Http\Kernel
class Kernel implements KernelContract
{
    protected function sendRequestThroughRouter($request)
    {
        $this->app->instance('request', $request);

        Facade::clearResolvedInstance('request');

        $this->bootstrap();

        return (new Pipeline($this->app))
                    ->send($request)
                    ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
                    ->then($this->dispatchToRouter());
    }

    protected function dispatchToRouter()
    {
        return function ($request) {
            $this->app->instance('request', $request);

            return $this->router->dispatch($request);
        };
    }

}
```
上面的代码可以看出`Pipeline`的`destination`就是`dispatchToRouter`函数返回的闭包:

也就是最终的目的地址是这样的一个闭包:
```php
$destination = function ($request) {
    $this->app->instance('request', $request);
    return $this->router->dispatch($request);
};
```
在闭包里调用了`router`的`dispatch`方法，路由寻址就发生在`dispatch`的一开始的`findRoute`里：

```php
class Router implements RegistrarContract, BindingRegistrar
{   
    public function dispatch(Request $request)
    {
        $this->currentRequest = $request;

        return $this->dispatchToRoute($request);
    }

    public function dispatchToRoute(Request $request)
    {
        return $this->runRoute($request, $this->findRoute($request));
    }

    protected function findRoute($request)
    {
        $this->current = $route = $this->routes->match($request);

        $this->container->instance(Route::class, $route);

        return $route;
    }
}
```
寻找路由的任务由 `RouteCollection` 负责，这个函数负责匹配路由，并且把 `request` 的 `url` 参数绑定到路由中：
```php
class RouteCollection implements Countable, IteratorAggregate
{
    public function match(Request $request)
    {
        $routes = $this->get($request->getMethod());

        $route = $this->matchAgainstRoutes($routes, $request);

        if (! is_null($route)) {
            //找到匹配的路由后，将URI里的路径参数绑定赋值给路由(如果有的话)
            return $route->bind($request);
        }

        $others = $this->checkForAlternateVerbs($request);

        if (count($others) > 0) {
            return $this->getRouteForMethods($request, $others);
        }

        throw new NotFoundHttpException;
    }

    protected function matchAgainstRoutes(array $routes, $request, $includingMethod = true)
    {
        return Arr::first($routes, function ($value) use ($request, $includingMethod) {
            return $value->matches($request, $includingMethod);
        });
    }
}

class Route
{
    public function matches(Request $request, $includingMethod = true)
    {
        $this->compileRoute();

        foreach ($this->getValidators() as $validator) {
            if (! $includingMethod && $validator instanceof MethodValidator) {
                continue;
            }

            if (! $validator->matches($this, $request)) {
                return false;
            }
        }

        return true;
    }
}
```

`$routes = $this->get($request->getMethod());`会先加载注册路由阶段在`RouteCollection`里生成的`routes`属性里的值，`routes`中存放了`HTTP`请求方法与路由对象的映射。

然后依次调用这堆路由里路由对象的`matches`方法， `matches`方法, `matches`方法里会对`HTTP`请求对象进行一些验证，验证对应的`Validator`是：`UriValidator`、`MethodValidator`、`SchemeValidator`、`HostValidator`。
在验证之前在`$this->compileRoute()`里会将路由的规则转换成正则表达式。

`UriValidator`主要是看请求对象的`URI`是否与路由的正则规则匹配能匹配上:

```php
class UriValidator implements ValidatorInterface
{
    public function matches(Route $route, Request $request)
    {
        $path = $request->path() == '/' ? '/' : '/'.$request->path();

        return preg_match($route->getCompiled()->getRegex(), rawurldecode($path));
    }
}
```

`MethodValidator`验证请求方法, `SchemeValidator`验证协议是否正确`(http|https)`, `HostValidator`验证域名, 如果路由中不设置`host`属性，那么这个验证不会进行

一旦某个路由通过了全部的认证就将会被返回，接下来就要将请求对象`URI`里的路径参数绑定复制给路由参数:

### 路由参数绑定

```php
class Route
{
    public function bind(Request $request)
    {
        $this->compileRoute();

        $this->parameters = (new RouteParameterBinder($this))
                        ->parameters($request);

        return $this;
    }
}

class RouteParameterBinder
{
    public function parameters($request)
    {
        $parameters = $this->bindPathParameters($request);

        if (! is_null($this->route->compiled->getHostRegex())) {
            $parameters = $this->bindHostParameters(
                $request, $parameters
            );
        }

        return $this->replaceDefaults($parameters);
    }

    protected function bindPathParameters($request)
    {
            preg_match($this->route->compiled->getRegex(), '/'.$request->decodedPath(), $matches);

            return $this->matchToKeys(array_slice($matches, 1));
    }

    protected function matchToKeys(array $matches)
    {
        if (empty($parameterNames = $this->route->parameterNames())) {
            return [];
        }

        $parameters = array_intersect_key($matches, array_flip($parameterNames));

        return array_filter($parameters, function ($value) {
            return is_string($value) && strlen($value) > 0;
        });
    }
}
```

赋值路由参数完成后路由寻址的过程就结束了，结下来就该运行通过匹配路由中对应的控制器方法返回响应对象了

```php
namespace Illuminate\Routing;
class Router implements RegistrarContract, BindingRegistrar
{   
    public function dispatch(Request $request)
    {
        $this->currentRequest = $request;

        return $this->dispatchToRoute($request);
    }

    public function dispatchToRoute(Request $request)
    {
        return $this->runRoute($request, $this->findRoute($request));
    }

    protected function runRoute(Request $request, Route $route)
    {
        $request->setRouteResolver(function () use ($route) {
            return $route;
        });

        $this->events->dispatch(new Events\RouteMatched($route, $request));

        return $this->prepareResponse($request,
            $this->runRouteWithinStack($route, $request)
        );
    }

    protected function runRouteWithinStack(Route $route, Request $request)
    {
        $shouldSkipMiddleware = $this->container->bound('middleware.disable') &&
                            $this->container->make('middleware.disable') === true;
    //收集路由和控制器里应用的中间件
        $middleware = $shouldSkipMiddleware ? [] : $this->gatherRouteMiddleware($route);

        return (new Pipeline($this->container))
                    ->send($request)
                    ->through($middleware)
                    ->then(function ($request) use ($route) {
                        return $this->prepareResponse(
                            $request, $route->run()
                        );
                    });

    }
}

namespace Illuminate\Routing;
class Route
{
    public function run()
    {
        $this->container = $this->container ?: new Container;
        try {
            if ($this->isControllerAction()) {
                return $this->runController();
            }
            return $this->runCallable();
        } catch (HttpResponseException $e) {
            return $e->getResponse();
        }
    }
}
```

最终会执行路由的`run`方法 当然也会判断是一个闭包还是一个控制器方法进行调用 最后将结果封装成`Response`  返回给客户端  当然这里只是简单的介绍了路由所经过的中间层 也就是中间件的执行逻辑  这个需要再去详细研究。