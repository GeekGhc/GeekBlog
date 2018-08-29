---
title: Laravel-VueJs 知乎
date: 2017-04-06
categories:
  - Laravel
tags:
    - laravel
    - vuejs
---
基本laravel5.3开发的知乎问答系统 采用AdminLTE后台模板

## Laravel-Zhihu
基于**Laravel5.3** 开发
后台模板采用[AdminlTE-For-Laravel](https://github.com/GeekGhc/adminLTE-for-laravel)

## 前言
基于[laravist](https://www.laravist.com/)社区知乎系列项目

> `Github` 项目地址  [laravel-zhihu](https://github.com/GeekGhc/zhihu-app)

## 功能

### 用户
-  登录注册(邮件认证)
-  用户设置
-  头像上传至七牛云存储
-  修改密码
-  忘记密码(邮件认证)
-  用户相互关注(邮件提醒)
-  用户发送私信(消息通知)
-  显示私信(已读和未读)
-  标志私信
-  标志私信全部已读
-  回复私信
-  个人主页(各项数据)

### 问题
-  问题列表
-  收藏问题(消息通知)
-  分享问题到第三方
-  问题答案评论(消息通知)
-  问题评论,答案评论(消息通知)
-  答案点赞
-  发布问题
-  修改问题(仅限问题的作者)
-  删除问题(仅限问题的作者)

### 后台
-  系统用户
-  系统用户资料修改,删除
-  管理员资料修改
-  创建问题(可置顶)
-  编辑问题(对任何用户的问题)
-  删除问题(对任何用户的问题)
-  问题评论
-  删除问题评论(对任何用户的评论)
-  文章标签
-  删除文章标签

## 效果预览
### 前台部分
![jekyll](/images/articles/2017-04-06/7.png)
![jekyll](/images/articles/2017-04-06/2.png)
![jekyll](/images/articles/2017-04-06/6.png)
![jekyll](/images/articles/2017-04-06/3.png)
### 后台部分
![jekyll](/images/articles/2017-04-06/8.png)
![jekyll](/images/articles/2017-04-06/9.png)

## 安装

> 前台部分为1.0版本 前后台为2.0版本

### 1.克隆源码到本地
> git clone https://github.com/GeekGhc/zhihu-app

### 2.进入项目目录
> cd zhihu-app

### 3. 拷贝`.env`文件
一些 `secret key` 改成自己服务的`key`即可
> cp .env.example .env

### 4. 下载相关的依赖包(也可以使用yarn)
下载`laravel`相关依赖的包
> composer install

### 5. 创建数据
> php artisan zhihu:install
