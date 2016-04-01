---
layout: post
title: Rails应用性能优化一例
tags: [Web Application, Performance Optimization, Ruby, Ruby on Rails, 性能优化]
---
{% include JB/setup %}

Rails是一款经典的Web开发框架，它的许多设计思想为使用者带来了很多开发上的便利。它非常适合于快速搭建Web应用程序，修改和维护的便利性使它受到许多创业型公司的青睐。然而过度地依赖其所带来的便利而不加节制地使用，往往会随着程序复杂度的增加，出现性能上的降低。对于大多数创业型公司的初级产品来说，还不值得动用复杂缓存机制或者迁移数据库来进行优化，此时针对Rails应用本身的一些低效代码进行优化，往往就能取得不错的效果。

下面举一个实际中遇到的性能优化案例。项目背景是一个小型电商系统，有许多种类的商品在站点上售卖，但是使用了一段时间后随着商品种类增多，发现页面加载缓慢，用户体验变差。

### 找出瓶颈
可以先用一些性能分析工具来测量网站的性能具体指标，例如：

* 全面的网站性能监测工具：[New Relic](http://newrelic.com)
* 浏览器的开发工具，如Chrome自带的Developer Tools
* 压力测试工具：[Apache ab](http://httpd.apache.org/docs/2.0/programs/ab.html)

通过这些工具可以直观地看出页面的平均响应时间，帮助你分析确定性能瓶颈。例如New Relic可以分析出一个完整的请求响应过程内，数据库查询、应用处理以及页面渲染等各个步骤各占用多长时间。还可以列举出最耗时的是哪些请求。

通过分析发现，页面的响应时间随着吞吐量增加而显著增加，而且主要耗费在应用处理过程中，而其他的数据库查询等耗时并没有显著增加。所以可以着手重点优化后台代码。

### 去除冗余
找出最耗时的一个请求进行优化，在本例中这个请求是一个用于获取商品分类信息的接口。商品类别分为两级，权且称为“大类”和“小类”，每一个大类下可以包含多个小类，但一个小类只能归属于一个大类。使用一个名为`Category`的Model表示一个分类，`categories`表中的`parent`字段用于表示小类记录中的父类id。

之前该接口的代码基本是这样的：

    class CategoriesController < ApplicationController
      def index
        categories = Category.parent_categories.sale_in.for_city(city_id).order('priority desc')
        render json: categories.map { |category| CategoryPresenter.new(category) }
      end
    end

基本逻辑是先查询出符合要求的categories，然后使用CategoryPresenter这样一个表示器，将数据转化为json形式。具体用法参见[roar](https://github.com/apotonick/roar)这个gem包的说明。

CategoryPresenter的代码是这样的：

    class CategoryPresenter < Roar::Decorator
      include Roar::JSON
      property :id
      property :name
      property :oss_url
      collection :children_categories, as: 'children', extend: CategoryPresenter
    end

其中用到了`children_categories`方法，这一方法用于查找该类别下的子类，在`Category`类中是这样定义的：

    def children_categories
      children = Category.where(parent: self.id, status: Category.statuses[:sale_in]).order('priority desc')
      children.presence || nil
    end

将这几部分的代码联系起来仔细分析下，并结合后台日志里输出的sql查询语句，就会发现存在一些不必要的冗余逻辑。`CategoryPresenter`这个表示器类中递归调用了自己，来对子分类`children_categories`迭代进行表示，直到子分类不再具有下一级子分类为止。然而，从业务上已知最多只有“大类”和“小类”两层分类，每个小类不会存在更低一层的子分类，所以实际上迭代结束的出口处，对每个小类再做一遍查询子分类的操作，是多余的。

那么实际上可以将`CategoryPresenter`根据大类和小类的不同用途拆分为不同的两个表示器`ParentCategoryPresenter`和`CategoryPresenter`，分别用于对大类和小类进行表示。这样一来对于小类，就退化为一个最简形式的表示器，而不用再去迭代查询子分类。

    class CategoryPresenter < Roar::Decorator
      include Roar::JSON
      property :id
      property :name
      property :oss_url
    end

    class ParentCategoryPresenter < CategoryPresenter
      collection :children_categories, as: 'children', extend: CategoryPresenter
    end

### 数据查询优化
再观察一下运行过程中产生的sql语句，可以发现有多次查询存在，原因是原来的控制器代码`CategoriesController`中存在一个循环，会对查出来的每个大类再分别进行一次查询子分类的操作，是一个典型的N+1次查询：

    categories = Category.parent_categories.sale_in.for_city(city_id).order('priority desc')
    render json: categories.map { |category| ParentCategoryPresenter.new(category) }

解决的办法是利用Rails Active Record中提供的`Includes`方法，将N+1次查询变为一次查询：

    categories = Category.parent_categories.sale_in.for_city(city_id).includes(:children_categories).order('priority desc')
    render json: categories.map { |category| ParentCategoryPresenter.new(category) }

### 对比结果
要知道每一步优化有没有起到效果，最好能够快速得到反馈结果。如果使用New Relic这样的网站性能监测工具去获取反馈，需要比较长的周期。那么本地开发时就可以使用[ab](http://httpd.apache.org/docs/2.0/programs/ab.html)这样的压测工具，来对被优化的接口快速进行一次性能测试。

在相同的测试条件下，优化前的测试结果是每次请求响应时间平均为54ms：

    Concurrency Level:      10
    Time taken for tests:   5.427 seconds
    Complete requests:      100
    Failed requests:        0
    Total transferred:      238500 bytes
    HTML transferred:       192900 bytes
    Requests per second:    18.43 [#/sec] (mean)
    Time per request:       542.674 [ms] (mean)
    Time per request:       54.267 [ms] (mean, across all concurrent requests)
    Transfer rate:          42.92 [Kbytes/sec] received

优化后的测试结果是每次请求响应时间平均为22ms：

    Concurrency Level:      10
    Time taken for tests:   2.256 seconds
    Complete requests:      100
    Failed requests:        0
    Total transferred:      238500 bytes
    HTML transferred:       192900 bytes
    Requests per second:    44.34 [#/sec] (mean)
    Time per request:       225.552 [ms] (mean)
    Time per request:       22.555 [ms] (mean, across all concurrent requests)
    Transfer rate:          103.26 [Kbytes/sec] received

从平均响应时间来看，性能得到了明显提升，已经足够满足当前需求。当然本例中只是针对Rails部分代码进行的优化，相信从其它角度入手，还有值得优化的空间。