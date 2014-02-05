---
layout: post
title: 使用Nginx+Unicorn+Capistrano+Sinatra+DataMapper搭建Ruby Web应用
tags : [Web Application, Ruby, Nginx, Unicorn, Capistrano, Sinatra, DataMapper]
---
{% include JB/setup %}

记录一下使用如下工具在[阿里云](http://www.aliyun.com/)上搭建一个简单的Ruby Web应用的过程：
###工具集：
* [Nginx](http://nginx.org/cn) --- 前端服务器
* [Unicorn](http://unicorn.bogomips.org) --- 应用容器
* [Capistrano](http://www.capistranorb.com) --- 自动部署工具
* [Sinatra](http://www.sinatrarb.com) --- 极轻量级Web框架
* [DataMapper](http://datamapper.org) --- ORM框架
* [MySQL](http://www.mysql.com) --- 持久化存储
***
###步骤：


1. 成员变量（实例变量）的声明：Java为静态声明，只要构造出对象就具有相同的实例变量；Ruby则为动态声明，不同的对象可能具有不同的实例变量，例如：

        def MyClass 
          
          def foo
            @v = 1
          end
        end 


***

代码：

    this is some code

