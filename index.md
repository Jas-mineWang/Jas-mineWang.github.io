---
layout: default
title: 首页
---

你好，我是 JasmineWang。

这里记录我在软件开发、问题排查和开源学习过程中的知识与经验。

我希望这些真实的工程记录，能帮助遇到类似问题的人少走一些弯路。

## 最新文章

{% if site.posts.size > 0 %}
<ul class="post-list">
  {% for post in site.posts %}
    <li>
      <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
      <h3>
        <a class="post-link" href="{{ post.url | relative_url }}">
          {{ post.title | escape }}
        </a>
      </h3>
      {% if post.excerpt %}
        {{ post.excerpt | strip_html | truncate: 120 }}
      {% endif %}
    </li>
  {% endfor %}
</ul>
{% else %}
暂无文章。
{% endif %}
