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
2. 新增模型类。假设要增加一个简单的功能，用户在页面上通过表单输入一个字符串作为关键字，每提交一次，应用就将该关键字记录下来存储到数据库中，模型类如下：
	
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
3. 新增模板文件。这里使用一个haml文件，包括一个简单的表单和提交按钮，供用户提交关键字。模板文件如下：

		# views/search.haml
		!!!
		%html
		  %head
		  %body
		    %h3 Search
		    %form.form{:action => '#', :method => 'get'}
		      %fieldset
		        %legend Keyword Input
		        %ul
		          %li
		            %input#keyword{:type => 'text', :name => 'q', :value => params[:q]}
		          %li
		            %input{:type => 'submit', :value => 'Search'}

4. 新增一个routes文件，用于响应表单提交请求：

		# routes/search.rb
		class App < Sinatra::Base

		  get '/search' do
		    q = params[:q]
		    History.create(:keyword => q) unless q.nil? or q.strip.empty?
		    haml :search
		  end

		end
	***
		# routes/init.rb
		require_relative 'search'
	另外`app.rb`中需`require_relative 'routes/init'`增加对routes文件的引入。
5. push代码到git仓库后使用`cap deploy`进行部署。在正式部署之前需先在远程主机上安装mysql，部署时如果有类似于以下的报错：

		  servers: ["115.28.137.21"]
		  [root@115.28.137.21] executing command
 		** [out :: root@115.28.137.21] Gem::Ext::BuildError: ERROR: Failed to build gem native extension.
		** [out :: root@115.28.137.21] 
		** [out :: root@115.28.137.21] checking for main() in -lmysqlclient... no
		......
		** [out :: root@115.28.137.21] make failed, exit code 2
		** [out :: root@115.28.137.21] 
		** [out :: root@115.28.137.21] Gem files will remain installed in /srv/dev-helper/shared/bundle/ruby/1.9.1/gems/do_mysql-0.10.13 for inspection.
		** [out :: root@115.28.137.21] Results logged to /srv/dev-helper/shared/bundle/ruby/1.9.1/extensions/x86_64-linux/1.9.1/do_mysql-0.10.13/gem_make.out
		** [out :: root@115.28.137.21] An error occurred while installing do_mysql (0.10.13), and Bundler cannot
		** [out :: root@115.28.137.21] continue.
		** [out :: root@115.28.137.21] Make sure that `gem install do_mysql -v '0.10.13'` succeeds before bundling.
	多是因为找不到mysql安装路径所致，可以在远程主机上运行：
		
		bundle config build.do_mysql --with-mysql-config=/usr/bin/mysql
	然后再进行部署。部署成功后就可以看到新功能生效了。
			
###参考：
* [Getting started with DataMapper](http://datamapper.org/getting-started.html)
* [Fixing bundle install/update of do_mysql on Mac OS X](https://gist.github.com/LeZuse/2409095)