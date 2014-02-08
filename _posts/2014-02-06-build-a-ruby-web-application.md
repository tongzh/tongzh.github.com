---
layout: post
title: 使用Nginx+Unicorn+Capistrano+Sinatra搭建Ruby Web应用
tags : [Web Application, Deploy, Ruby, Nginx, Unicorn, Capistrano, Sinatra]
---
{% include JB/setup %}

记录一下使用如下工具搭建一个简单的Ruby Web应用的过程：
###工具集：
* [Nginx](http://nginx.org) --- 前端服务器
* [Unicorn](http://unicorn.bogomips.org) --- 应用容器
* [Capistrano](http://www.capistranorb.com) --- 自动部署工具
* [Sinatra](http://www.sinatrarb.com) --- 极轻量级Web框架

###步骤：
1. 使用Sinatra提供的DSL编写一个能够运行的最小化web应用：

		# Gemfile
		source 'http://ruby.taobao.org/'
		gem 'sinatra'
	***
		# app.rb
		require 'sinatra'
		
		get '/' do
		  'Hello world'
		end
	使用`bundle exec ruby app.rb`检验应用是否能正常运行：
	
		twer@app$ bundle exec ruby app.rb
		[2014-02-06 03:06:59] INFO  WEBrick 1.3.1
		[2014-02-06 03:06:59] INFO  ruby 1.9.3 (2013-11-22) [x86_64-darwin13.0.0]
		== Sinatra/1.4.4 has taken the stage on 4567 for development with backup from WEBrick
		[2014-02-06 03:06:59] INFO  WEBrick::HTTPServer#start: pid=62687 port=4567
	访问[http://localhost:4567](http://localhost:4567)应该能得到'Hello world'的响应
2. 现在将其整理为Rack-based app：
		
		# app.rb
		require 'sinatra'
		
		class App < Sinatra::Base
  		  get '/' do
    	    'Hello world'
  		  end
		end
	***
		# config.ru
		root = ::File.dirname(__FILE__)
		require ::File.join(root, 'app')
		
		run App
	使用`rackup -p1818`检验应用是否能正常运行：
	
		twer@app$ rackup -p1818
		Thin web server (v1.6.1 codename Death Proof)
		Maximum connections set to 1024
		Listening on 0.0.0.0:1818, CTRL+C to stop
	访问[http://localhost:1818](http://localhost:1818)应该能得到'Hello world'的响应
3. 引入unicorn和capistrano配置：

		# Gemfile
		source 'http://ruby.taobao.org/'
		gem 'sinatra'
		gem 'unicorn'
		gem 'capistrano', '~> 2.15.5'
		gem 'rvm-capistrano'
	使用`bundle install`安装相应的gem包后，运行`capify .`生成部署所需的配置文件：
		
		twer@app$ capify .
		[add] writing './Capfile'
		[add] making directory './config'
		[add] writing './config/deploy.rb'
		[done] capified!
	配置config/deploy.rb，注意git仓库地址的配置，以及阿里云服务器ip地址及用户名的配置：
		
		# config/deploy.rb
		require 'rvm/capistrano'
		set :rvm_ruby_string, '1.9.3'
		set :rvm_type, :system
		
		# Bundler tasks
		require 'bundler/capistrano'
		
		# Your application name
		set :application, 'my-ruby-web-demo'
		
		# Input your git repository address
		set :repository,  'git@github.com:tongzh/my-ruby-web-demo.git'
		
		set :scm, :git
		
		# do not use sudo
		set :use_sudo, false
		set(:run_method) { use_sudo ? :sudo : :run }
		
		# This is needed to correctly handle sudo password prompt
		default_run_options[:pty] = true
		
		# Input your username to login remote server address
		set :user, 'root'
		set :group, user
		set :runner, user
		
		# Input your server address
		set :host, "#{user}@115.28.137.21"
		role :web, host
		role :app, host

		set :rails_env, :production

		# Where will it be located on a server?
		set :deploy_to, "/srv/#{application}"
		set :unicorn_conf, "#{deploy_to}/current/config/unicorn.rb"
		set :unicorn_pid, "#{deploy_to}/shared/pids/unicorn.pid"
		
		# Unicorn control tasks
		namespace :deploy do
  		  task :restart do
    	    run "if [ -f #{unicorn_pid} ]; then kill -USR2 `cat #{unicorn_pid}`; else cd #{deploy_to}/current && bundle exec unicorn -c #{unicorn_conf} -E #{rails_env} -D; fi"
	      end
  		  task :start do
    	    run "cd #{deploy_to}/current && bundle exec unicorn -c #{unicorn_conf} -E #{rails_env} -D"
		  end
  		  task :stop do
    	    run "if [ -f #{unicorn_pid} ]; then kill -QUIT `cat #{unicorn_pid}`; fi"
	      end
		end
	新建config/unicorn.rb文件，对unicorn进行配置：
	
		# config/unicorn.rb
		deploy_to = '/srv/my-ruby-web-demo'
		rails_root = "#{deploy_to}/current"
		pid_file = "#{deploy_to}/shared/pids/unicorn.pid"
		socket_file= "#{deploy_to}/shared/unicorn.sock"
		log_file = "#{rails_root}/log/unicorn.log"
		err_log = "#{rails_root}/log/unicorn_error.log"
		old_pid = pid_file + '.oldbin'

		timeout 30
		worker_processes 2 # increase or decrease
		listen socket_file, :backlog => 1024

		pid pid_file
		stderr_path err_log
		stdout_path log_file

		# make forks faster
		preload_app true

		# make sure that Bundler finds the Gemfile
		before_exec do |server|
		  ENV['BUNDLE_GEMFILE'] = File.expand_path('../Gemfile', File.dirname(__FILE__))
		end

		before_fork do |server, worker|
		  defined?(ActiveRecord::Base) and
		    ActiveRecord::Base.connection.disconnect!

		  # zero downtime deploy magic:
		  # if unicorn is already running, ask it to start a new process and quit.
		  if File.exists?(old_pid) && server.pid != old_pid
		    begin
		      Process.kill("QUIT", File.read(old_pid).to_i)
		    rescue Errno::ENOENT, Errno::ESRCH
		      # someone else did our job for us
		    end
		  end
		end

		after_fork do |server, worker|

		  # re-establish activerecord connections.
		  defined?(ActiveRecord::Base) and
		    ActiveRecord::Base.establish_connection
		end
	然后push代码到git仓库。
4. 在正式进行部署前，ssh登录到远程主机上依次安装所需的软件环境：
	* [安装RVM和Ruby](http://ruby-china.org/wiki/install_ruby_guide)
	* [安装git](http://git-scm.com/download/linux)
	* [添加github ssh key](https://help.github.com/articles/generating-ssh-keys)
5. 依次使用`cap deploy:setup`和`cap deploy`命令进行部署：
		
		twer@app<master>$ cap deploy:setup
		    triggering load callbacks
		  * 2014-02-06 21:13:12 executing `deploy:setup'
		  * executing "mkdir -p /srv/my-ruby-web-demo /srv/my-ruby-web-demo/releases /srv/my-ruby-web-demo/shared /srv/my-ruby-web-demo/shared/system /srv/my-ruby-web-demo/shared/log /srv/my-ruby-web-demo/shared/pids"
		    servers: ["115.28.137.21"]
		Password: 
		    [root@115.28.137.21] executing command
		    command finished in 647ms
		  * executing "chmod g+w /srv/my-ruby-web-demo /srv/my-ruby-web-demo/releases /srv/my-ruby-web-demo/shared /srv/my-ruby-web-demo/shared/system /srv/my-ruby-web-demo/shared/log /srv/my-ruby-web-demo/shared/pids"
		    servers: ["115.28.137.21"]
		    [root@115.28.137.21] executing command
		    command finished in 644ms
	***
		twer@app<master>$ cap deploy
		    triggering load callbacks
		  * 2014-02-06 21:20:06 executing `deploy'
		  * 2014-02-06 21:20:06 executing `deploy:update'
		 ** transaction: start
		  * 2014-02-06 21:20:06 executing `deploy:update_code'
		    executing locally: "git ls-remote git@github.com:tongzh/my-ruby-web-demo.git HEAD"
		    command finished in 7722ms
		  * executing "git clone -q git@github.com:tongzh/my-ruby-web-demo.git /srv/my-ruby-web-demo/releases/20140206102014 && cd /srv/my-ruby-web-demo/releases/20140206102014 && git checkout -q -b deploy 0de5794f2d7e13af76e671df6a7796c676e88e2b && (echo 0de5794f2d7e13af76e671df6a7796c676e88e2b > /srv/my-ruby-web-demo/releases/20140206102014/REVISION)"
		    servers: ["115.28.137.21"]
		Password: 
		    [root@115.28.137.21] executing command
		    command finished in 4891ms
		  * 2014-02-06 21:20:25 executing `deploy:finalize_update'
		    triggering before callbacks for `deploy:finalize_update'
		  * 2014-02-06 21:20:25 executing `bundle:install'
		  * executing "cd /srv/my-ruby-web-demo/releases/20140206102014 && bundle install --gemfile /srv/my-ruby-web-demo/releases/20140206102014/Gemfile --path /srv/my-ruby-web-demo/shared/bundle --deployment --quiet --without development test"
		    servers: ["115.28.137.21"]
		    [root@115.28.137.21] executing command
		    command finished in 45056ms
		  * executing "chmod -R -- g+w /srv/my-ruby-web-demo/releases/20140206102014 && rm -rf -- /srv/my-ruby-web-demo/releases/20140206102014/public/system && mkdir -p -- /srv/my-ruby-web-demo/releases/20140206102014/public/ && ln -s -- /srv/my-ruby-web-demo/shared/system /srv/my-ruby-web-demo/releases/20140206102014/public/system && rm -rf -- /srv/my-ruby-web-demo/releases/20140206102014/log && ln -s -- /srv/my-ruby-web-demo/shared/log /srv/my-ruby-web-demo/releases/20140206102014/log && rm -rf -- /srv/my-ruby-web-demo/releases/20140206102014/tmp/pids && mkdir -p -- /srv/my-ruby-web-demo/releases/20140206102014/tmp/ && ln -s -- /srv/my-ruby-web-demo/shared/pids /srv/my-ruby-web-demo/releases/20140206102014/tmp/pids"
		    servers: ["115.28.137.21"]
		    [root@115.28.137.21] executing command
		    command finished in 692ms
		  * executing "find /srv/my-ruby-web-demo/releases/20140206102014/public/images /srv/my-ruby-web-demo/releases/20140206102014/public/stylesheets /srv/my-ruby-web-demo/releases/20140206102014/public/javascripts -exec touch -t 201402061021.11 -- {} ';'; true"
		    servers: ["115.28.137.21"]
		    [root@115.28.137.21] executing command
		 ** [out :: root@115.28.137.21] find: `/srv/my-ruby-web-demo/releases/20140206102014/public/images': No such file or directory
		 ** [out :: root@115.28.137.21] find: `/srv/my-ruby-web-demo/releases/20140206102014/public/stylesheets': No such file or directory
		 ** [out :: root@115.28.137.21] find: `/srv/my-ruby-web-demo/releases/20140206102014/public/javascripts': No such file or directory
		    command finished in 655ms
		  * 2014-02-06 21:21:12 executing `deploy:create_symlink'
		  * executing "rm -f /srv/my-ruby-web-demo/current &&  ln -s /srv/my-ruby-web-demo/releases/20140206102014 /srv/my-ruby-web-demo/current"
		    servers: ["115.28.137.21"]
		    [root@115.28.137.21] executing command
		    command finished in 702ms
		 ** transaction: commit
		  * 2014-02-06 21:21:12 executing `deploy:restart'
		  * executing "if [ -f /srv/my-ruby-web-demo/shared/pids/unicorn.pid ]; then kill -USR2 `cat /srv/my-ruby-web-demo/shared/pids/unicorn.pid`; else cd /srv/my-ruby-web-demo/current && bundle exec unicorn -c /srv/my-ruby-web-demo/current/config/unicorn.rb -E production -D; fi"
		    servers: ["115.28.137.21"]
		    [root@115.28.137.21] executing command
		    command finished in 1633ms
	确保以上操作无异常输出，然后登录到远程主机上查看应用是否已正常装载到unicorn进程中：

		root@~$ ps aux | grep my-ruby-web-demo
		root     20071  0.0  7.3 218396 37016 ?        Sl   21:21   0:00 unicorn master -c /srv/my-ruby-web-demo/current/config/unicorn.rb -E production -D                                                 
		root     20074  0.0  7.2 218396 36476 ?        Sl   21:21   0:00 unicorn worker[0] -c /srv/my-ruby-web-demo/current/config/unicorn.rb -E production -D                                              
		root     20076  0.0  7.2 218396 36476 ?        Sl   21:21   0:00 unicorn worker[1] -c /srv/my-ruby-web-demo/current/config/unicorn.rb -E production -D                                              
		root     20141  0.0  0.1 103208   824 pts/1    S+   21:23   0:00 grep my-ruby-web-demo
6. 在远程主机上[安装nginx](http://nginx.org/en/docs/install.html)，并在nginx配置中增添一个反向代理配置，将请求分发给应用所在的unicorn进程：
		
		# /etc/nginx/default.conf
		upstream my-ruby-web-demo-unicorn {
		    server unix:/srv/my-ruby-web-demo/shared/unicorn.sock fail_timeout=0;
		}
 
		server {
		    listen       80;
		    server_name  localhost;

		    root /srv/my-ruby-web-demo/current/public;
 
		    location / {
		        try_files $uri @net;
		    }
 
		    location @net {
		        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		        proxy_set_header Host $http_host;
		        proxy_redirect off;
		        proxy_pass http://my-ruby-web-demo-unicorn;
		    }

		    error_page  404              /404.html;

		    # redirect server error pages to the static page /50x.html
		    error_page   500 502 503 504  /50x.html;
		    location = /50x.html {
		        root   /usr/share/nginx/html;
		    }
		}
	在远程主机上重启nginx：`service nginx restart`，之后通过远程主机的ip就可以访问到所部署的应用程序的'Hello world'响应了。
			
		
###参考：
* [Deploying With Sinatra + Capistrano + Unicorn](http://tech.tulentsev.com/2012/03/deploying-with-sinatra-capistrano-unicorn/)
* [Capistrano + Nginx + Unicorn + Sinatra on Ubuntu](https://gist.github.com/wlangstroth/3740923)