---
title: 聊聊PHP中实现的依赖注入(Di)实现
date: 2018-10-28
categories:
  - PHP
tags:
    - php
---
在`PHP`的开发中 利用`IOC`容器可以很方便的储存以及获取资源 这样就可以实现解耦, 使用依赖注入的好处就是有效的分离了对象和他所需的外部资源，使得他们松散耦合,这样就有利于功能的复用。这样一来程序代码的整体结构也会变的整节可读。即随取随用。

在开发的过程中,如果我们需要一个对象就需要不断的new一个新的对象,可能会在很多处地方用到这个对象的功能,这样的话如果一些的不好的编程习惯会造成对象的无法回收,这会为
整个项目的代码和维护造成不小的困扰,所以依照我们提倡的松耦合、少入侵的原则,可以采用`ICO`容器也就是对这些对象的集中管理。

这里我们可以看看`EasySwoole`里面是怎么实现这样的容器管理的

## 代码入手
首先为了实现容器的统一管理,定义了一个单例`trait`这趟就很好的对`Di`这个容器进行单例化。
```
trait Singleton
{
    private static $instance;

    static function getInstance(...$args)
    {
        if(!isset(self::$instance)){
            self::$instance = new static(...$args);
        }
        return self::$instance;
    }
}
```
实现代码很简单  就是首先定义一个私有事例  在暴露一个`static`获取事例的`function` 如果已实例化则直接返回 如果没有则在初始化的时候`new`一个对象返回

接下来就是`Di`这个容器的编写 `use`这个单例`trait`时它成为单例类  然后可以想象一下容器的设置/获取这样的一些操作 那么很自然的想到我们可以这么写
```
class Di
{
    use Singleton;
    private $container = array();

    public function set($key, $obj,...$arg):void
    {
        $this->container[$key] = array(
            "obj"=>$obj,
            "params"=>$arg,
        );
    }

    function delete($key):void
    {
        unset( $this->container[$key]);
    }

    function clear():void
    {
        $this->container = array();
    }

    /**
     * @param $key
     * @return null
     * @throws \Throwable
     */
    function get($key)
    {
        if(isset($this->container[$key])){
            $obj = $this->container[$key]['obj'];
            $params = $this->container[$key]['params'];
            if(is_object($obj) || is_callable($obj)){
                return $obj;
            }else if(is_string($obj) && class_exists($obj)){
                try{
                    $this->container[$key]['obj'] = new $obj(...$params);
                    return $this->container[$key]['obj'];
                }catch (\Throwable $throwable){
                    throw $throwable;
                }
            }else{
                return $obj;
            }
        }else{
            return null;
        }
    }
}
```
这里定义了四个`function` 分别是对象的设置、获取、删除和清空。首先可以看下set也就是注入函数的原型,它接收三个参数，第一个就是`key`名 也就是我们再项目其他地方需要用到时可以通过`key`来获取这个`object`。在set这个`function`中第二个参数`$obj`即为需要设置储存的`object`。第三个参数就是一系列参数，这个参数就是`object`需要初始化的参数。
所以在设置时同意放入私有的`container`变量中 因为这个是单例类 我们不需要暴露这个`container`变量 我们可以通过`get`获取这个`obj`

在`get`获取注入的资源时 首先进行判空  如果存在那么接下来就是对传之前传入对象的形式进行初始化`new`一个对象返回。也就是说如果是一个可调用的对象的话比如 `new User()` 那么直接返回这个`obj`。如果是一个`string`类型，并且存在这个`class`的话 很简单实例化这个类，并通过`set`时候传入放入初始化参数进行初始化对象并返回。另外其他的删除以及清空`container`的都比较简单。

在项目使用时如果需要获取这个对象那么可以直接通过`get`方法获取并调用  比如
```
$di = Di::getInstance();
$obj = $di->get('database');
```

那么就可以使用`key`为`database`的这个`object`。因为`Di`是单利类 所以对象的存取就会很好实现。 这里只是一个比较简单的`IOC`的实现 我们还可以根据服务的一些特性去实现其他的功能 具体的可以参考`Laravel`里的`IOC`的实现 有兴趣的可以了解一下。