---
title: Kafka的实践和使用
date: 2018-10-24
categories:
  - Kafka
tags:
    - kafka
---
`kafka`作为一个分布式的流处理平台 在实际的项目中有这广泛的应用 而其可以用于两大类别的应用分类
1. 构造实时流数据管道，它可以在系统或应用之间可靠地获取数据。 (相当于`message queue`)
2. 构建实时流式应用程序，对这些流数据进行转换或者影响。 (就是流处理，通过`kafka stream topic`和`topic`之间内部进行变化)

这里我们以`php`作为后端语言为例 引入`kafka`来解决一些实际问题

### PHP引入kafka
本地环境为`mac`环境 
首先通过`brew`安装工具
```shell
$ brew install kafka 
```
安装的同时会自动安装依赖`zookeeper`

而在安装`kafka`的扩展之前，在安装`php-rdkafka`之前，需要先安装`librdkafka`
```shell
$ git clone https://github.com/edenhill/librdkafka.git
$ cd  librdkafka
$ ./configure
$ make && make install
```
开始安装`kafka`扩展
```shell
$ git clone https://github.com/arnaud-lb/php-rdkafka.git
$ cd php-rdkafka
$ phpize
$ ./configure  --with-php-config=/usr/local/php/bin/php-config  ###你安装的php下的php-config路径
$ make && make install
```
编译完成之后再`php.ini`中加入扩展 `extension=rdkafaka.so`

### kafka启动执行
进入`mac`环境下的`kafka`编译后的目录  可以看到`kafka`和`zookeeper`的配置文件 `server.proprties`和`zookeeper.properties`
![1](/images/articles/2018-10-24/1.png)

启动`zookeeper`
```shell
$ zookeeper-server-start /usr/local/etc/kafka/zookeeper.properties
```
![1](/images/articles/2018-10-24/2.png)
启动服务后 端口是绑定到了**2181**端口

启动`kafaka`
进入服务的文件目录 可以看到关于`kafka`的一些命令列表 其中包含了镜像管理和日志管理
![1](/images/articles/2018-10-24/3.png)
```shell
$ kafka-server-start /usr/local/etc/kafka/server.properties
```

其实有关`kafka`的基本概念可以查看其官方文档的一些说明

在这里为了测试可以先创建一个`topic` 当然可以先看一下所有的`topics`
```shell
$ kafka-topics --list --zookeeper localhost:2181
```
![1](/images/articles/2018-10-24/4.png)
让我们创建一个名为`“test”`的`topic`，它有**3**个分区和1个副本
```shell
$ kafka-topics --create --zookeeper localhost:2181 --replication-factor 1 --partitions 3 --topic test
```
再次查看所有`topics`信息
![1](/images/articles/2018-10-24/5.png)

查看刚创建的话题信息
![1](/images/articles/2018-10-24/6.png)

启动生产者
```shell
$ kafka-console-producer --broker-list localhost:9092 --topic test
```
> 这里的9092是kafaka的端口地址 并非zookeeper端口地址

启动消费者
```shell
$ kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```
> 这里的from-beginning 表示信息每次都会从头开始读取

这里只是设置了单个服务  作为一个分布式处理平台  可以设置多个代理
![1](/images/articles/2018-10-24/7.png)

这里是另外两个服务的配置信息  也就是`kafka`的配置文件的信息
```shell
server-1.properties:
    broker.id=1
   zookeeper.connect=localhost:2181
    listeners=PLAINTEXT://:9093
    log.dir=/usr/local/var/lib/kafka-logs-1
 
server-2.properties:
    broker.id=2
    zookeeper.connect=localhost:2181
    listeners=PLAINTEXT://:9094
    log.dir=/usr/local/var/lib/kafka-logs-2
```
这样的话去启动其他两个新的节点
```shell
$ bin  kafka-server-start /usr/local/etc/kafka/server-1.properties
$ bin  kafka-server-start /usr/local/etc/kafka/server-2.properties
```
现在创建副本为**3**的新`topic`
```shell
$ kafka-topics --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-topic
```
现在已经有了一个集群 查看代理情况
![1](/images/articles/2018-10-24/8.png)

值得注意的是:
`“leader”`是负责给定分区所有读写操作的节点。每个节点都是随机选择的部分分区的领导者。
`“replicas”`是复制分区日志的节点列表，不管这些节点是`leader`还是仅仅活着。
`“isr”`是一组`“同步”replicas`，是`replicas`列表的子集，它活着并被指到`leader`。

## 相关链接
[Kafka 中文文档 - ApacheCN](http://kafka.apachecn.org)
[Apache Kafka](http://kafka.apache.org/documentation/)




