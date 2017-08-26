---
layout: page
title: Tong's Blog
---
{% include JB/setup %}


## Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


## Presentations

<ul class="posts">
  <li><span>19 Aug 2017 @ DevOpsDays Shanghai</span> &raquo; <a href="presentations/devopsdays_shanghai/msa_test_practices.pdf" target="_blank">轻量化微服务测试实践</a></li>
  <li><span>23 Apr 2017 @ DevOps MeetUp Xi'an</span> &raquo; <a href="presentations/pact_test/index.html" target="_blank">基于PACT的微服务契约测试</a></li>
</ul>

## Readings

<ul class="posts">
  <li><a href="https://readings.tongzh.top/practical-devops.html" target="_blank">DevOps实践：驭DevOps之力强化技术栈并优化IT运行</a></li>
  <li><a href="https://readings.tongzh.top/continuous-integration.html" target="_blank">持续集成：软件质量改进和风险降低之道</a></li>
  <li><a href="https://readings.tongzh.top/how-google-tests-software.html" target="_blank">Google软件测试之道</a></li>
  <li><a href="https://readings.tongzh.top/how-to-read-a-book.html" target="_blank">如何阅读一本书</a></li>
</ul>
