---
title: PHP的生成器,yield和协程
date: 2018-08-24
categories:
  - PHP
tags:
    - php
---
在很多的应用场景下 我们都需要基于PHP中的协程完成业务场景。而对于生成器和与之对应的yield关键字之前只有所耳闻 最近因此也想记录一下谈谈
自己的理解

## 迭代器
迭代最先接触的时候是在学习C++时  而迭代就是反复执行的一个过程 我们最常见的就是一个循环的执行流程
```
$arr = [1, 2, 3, 4, 5,6];

foreach($arr as $key => $value) {
    echo $key . ' => ' . $value . "\n";
}
```
就是这样的一个`foreach()`执行的便利流程 对目标数组进行内容的输出 每次迭代都会将数组的元素赋予`$value` 数组指针也会自动移动下一个元素位置直至全部遍历
像这样能够让外部的函数迭代自己内部数据的接口就是迭代器接口，对应的那个被迭代的自己就是迭代器对象

在PHP中迭代器的接口是统一的：
```
Iterator extends Traversable {

    // 返回当前的元素
    abstract public mixed current(void)
    // 返回当前元素的键
    abstract public scalar key(void)
    // 向下移动到下一个元素
    abstract public void next(void)
    // 返回到迭代器的第一个元素
    abstract public void rewind(void)
    // 检查当前位置是否有效
    abstract public boolean valid(void)
}
```
通过提供给我们的迭代器的接口我们就可以自己去实现自己想要的迭代器的功能
```
class MyIterator implements Iterator {
    private $position = 0;
    private $arr = [
        'first', 'second', 'third',
    ];
    public function __construct() {
        $this->position = 0;
    }
    public function rewind() {
        var_dump(__METHOD__);
        $this->position = 0;
    }
    public function current() {
        var_dump(__METHOD__);
        return $this->arr[$this->position];
    } 
    public function key() {
        var_dump(__METHOD__);
        return $this->position;
    }
    public function next() {
        var_dump(__METHOD__);
        ++$this->position;
    } 
    public function valid() {
        var_dump(__METHOD__);
        return isset($this->arr[$this->position]);
    }   
}
$it = new MyIterator();

foreach($it as $key => $value) {
    echo "\n";
    var_dump($key, $value);
}
```

## 协程
协程的支持是在迭代生成器的基础之上的 增加了回传数据给生成器的功能