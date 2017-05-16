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
  <li><span>23 Apr 2017 @ DevOps MeetUp Xi'an</span> &raquo; <a href="presentations/pact_test/index.html" target="_blank">基于PACT的微服务契约测试</a></li>
</ul>

## Readings

<ul class="posts">
  <li><a href="https://readings.tongzh.top" target="_blank">个人阅读笔记</a></li>
</ul>
