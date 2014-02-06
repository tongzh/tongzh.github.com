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
1. 使用Sinatra提供的DSL编写一个能够运行的最小化web应用

		# Gemfile
		gem 'sinatra'
		
		# app.rb
		require 'sinatra'
		
		get '/' do
		  "Hello world"
		end
	在代码所在路径中使用`bundle exec ruby app.rb`检验该应用是否能正常运行：
	
		twer@app$ bundle exec ruby app.rb
		[2014-02-06 03:06:59] INFO  WEBrick 1.3.1
		[2014-02-06 03:06:59] INFO  ruby 1.9.3 (2013-11-22) [x86_64-darwin13.0.0]
		== Sinatra/1.4.4 has taken the stage on 4567 for development with backup from WEBrick
		[2014-02-06 03:06:59] INFO  WEBrick::HTTPServer#start: pid=62687 port=4567
	现在访问[http://localhost:4567](http://localhost:4567)应该能得到'hello world'的响应
2. 现在将其整理为Rack-based app：