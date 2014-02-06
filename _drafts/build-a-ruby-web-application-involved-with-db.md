---
layout: post
title: 使用MySQL+DataMapper为Ruby Web应用增加数据库存储
tags : [Web Application, Deploy, Ruby, MySQL, DataMapper]
---
{% include JB/setup %}

在之前搭建的[简单Ruby Web应用](/2014/02/06/build-a-ruby-web-application/)基础上增加数据库存储：
###工具集：
* [DataMapper](http://datamapper.org) --- ORM框架
* [MySQL](http://www.mysql.com) --- 数据库

###步骤：
1. 引入data_mapper、dm-mysql-adapter、dm-migrations、haml等gem，并`bundle install`安装：

		# Gemfile
		source 'http://ruby.taobao.org/'
		gem 'sinatra'
		gem 'unicorn'
		gem 'capistrano', '~> 2.15.5'
		gem 'rvm-capistrano'
		gem 'data_mapper'
		gem 'dm-mysql-adapter'
		gem 'dm-migrations'
		gem 'haml'
2. 增加模型类。假设要增加一个简单的功能，用户在页面上通过表单输入一个字符串作为关键字，每提交一次，应用就将该关键字记录下来存储到数据库中，模型类如下：
	
		# models/history.rb
		class History
		  include DataMapper::Resource

		  property :id,               Serial
		  property :keyword,          String
		  property :created_at,       DateTime, :default => lambda { |r,p| Time.now }

		end
	增加一段连接MySQL以及建立对象-关系映射的初始化代码，注意：
	* `DataMapper.finalize`要在所有的模型类都已被引入之后调用
	* `DataMapper.auto_upgrade!`执行时会自动根据模型类对数据库进行migration升级操作，与`DataMapper.auto_migrate!`方法不同的是，`auto_upgrade!`仅会更新数据库表结构而不抹除已有数据，`auto_migrate!`则会调整表结构并抹除已有数据。
	***
	
		# models/init.rb
		require 'data_mapper'
		require 'dm-migrations'
		require_relative 'history'

		# A MySQL connection:
		DataMapper.setup(:default, 'mysql://root@localhost/test') # Your mysql username and database
		DataMapper.finalize
		DataMapper.auto_upgrade!
	并在app.rb中引入models/init.rb文件：
	
		# app.rb
		require 'sinatra'

		class App < Sinatra::Base
		  get '/' do
		    'Hello world'
		  end
		end

		require_relative 'models/init'

	
			
		
###参考：
* [Getting started with DataMapper](http://datamapper.org/getting-started.html)