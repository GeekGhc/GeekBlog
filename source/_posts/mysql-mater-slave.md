---
title: Mysql数据库的主从复制和读写分离
date: 2018-10-02
categories:
  - SQL
tags:
    - mysql
    - master
    - slave
---

数据库的应用在日常的工作和生产中充当着数据管理者的身份  对于一些企业级的数据 数据安全和数据的操作效率显得尤为重要

那么在这里将以数据库常见的主从复制和数据库的读写分离进行一个总结

## 读写分离
对于这样的一个应用首先要明白他的应用场景 我们知道`sql`的查询比一些新增 删除这些操作更为耗时

也就是主数据库负责网站`NonQuery`操作，从服务器负责`Query`操作，用户可以根据网站功能模特性块固定访问`Slave`服务器，或者自己写个池或队列，自由为请求分配从服务器连接

大型网站对于网站的大并发量的访问 除了实现网站的负载均衡 对于数据的处理也格外注意 如果还是传统的数据库架构 如果多的数据的操作势必会造成互数据的压力倍增从而导致请求超时和用户
体验的下降 因此我们会想到减少数据库的连接 除了代码的优化以及一些缓存技术的应用我们也是可以尝试这样的架构方式来减轻主数据库的压力  即主数据库负责写 从数据库只负责数据查询


## 主从复制
数据库的主从复制可以理解为就是主数据库也就是我们项目直接使用操作的数据库  而从数据库当主数据库进行数据操作变化时也会
同步更新 即主数据库怎么做从属数据库就怎么做 这样一来对于主数据库的数据就可以起到一种备份的作用 

对于原理其实就是对于主数据库在进行数据操作 如增删改查时 会把这些操作记录在二进制日志中 这样从属数据库如果拿到这份日志
这样就可以重复执行这些动作  所以也就达到了复制的作用

为了演示可以准备两台服务器  这里我用的是[DigitalOcean](https://cloud.digitalocean.com) 作为测试服务器很是方便 

选择创建两个`Droplets` 具体配置根据自己的需求来
![first](/images/articles/2018-10-02/01.png)
![first](/images/articles/2018-10-02/02.png)

这里由于选择本地的`ssh key`因此可以无需密码登录服务器

登录`master`和`slave`服务器安装`mysql` 这里我以**5.7**

```shell
$ apt-get install  mysql-server-5.7 -y
```
### 配置Mysql
#### Master
进入`mysql`的配置路径`/etc/mysql`
![first](/images/articles/2018-10-02/03.png)

这里更改`bind_ip`为内网`ip`
![first](/images/articles/2018-10-02/04.png)

更改`server_id=1`这里的1可以根据需要更改为一个数字一用来区分

最终配置为
```shell
server-id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log
expire_logs_days        = 10
max_binlog_size         = 100M
```

重启`mysql`服务
```shell
$ service mysql restart
```

进入`mysql`创建一个数据库`app`
```shell
$ create database app charset utf8mb4;
```

配置主数据
```shell
mysql> create user 'slave'@'10.138.12.79' identified by 'slavepwd';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'slave'@'10.138.12.79';
mysql> FLUSH PRIVILEGES;
```
我们可以查看到主数据库的状态
```shell
mysql> show master status;
```
![first](/images/articles/2018-10-02/05.png)

记录下这个日志的`position`和`file`

#### Slave
进入主服务器
我们需要配置下`mysql`的基本配置 路径还是`/etc/mysql`

需要区分从数据库的`server_id`这里更改为**2**
最终的配置为
```shell
server-id               = 2
log_bin                 = /var/log/mysql/mysql-bin.log
expire_logs_days        = 10
max_binlog_size         = 100M
```

接下来就是从数据库的配置 进入`mysql`
首先关系从属配置
```shell
mysql> stop slave;
```

```shell
mysql> CHANGE MASTER TO 
mysql> MASTER_HOST='10.138.204.172', 
mysql> MASTER_USER='slave', 
mysql> MASTER_PASSWORD='slavepwd', 
mysql> MASTER_LOG_FILE='mysql-bin.000001', 
mysql> MASTER_LOG_POS=771;
```
这里的`log`就是`master`数据库配置的信息

重启`slave`
```shell
mysql> start slave;
```
为了确认可以查看最终的`Slave`状态
```shell
mysql> show slave status/G
```
如果`Slave_IO_Running`和`Slave_SQL_Running`都为`yes`即为成功配置
![first](/images/articles/2018-10-02/06.png)

这样的话在主数据库同一个数据库下的`sql`都将同步到`salve`  也就实现了我们需要的`mysql`主从复制





