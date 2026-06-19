---
layout: page
title: Spring
permalink: /spring/
---

{% assign spring_posts = site.categories.spring %}
{% if spring_posts and spring_posts != empty %}
<ul class="post-list">
  {% for post in spring_posts %}
  <li>
    <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
    <h3 style="margin:.2em 0;">
      <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
    </h3>
    {% if post.description %}<p>{{ post.description | escape }}</p>{% endif %}
  </li>
  {% endfor %}
</ul>
{% else %}
<p>아직 Spring 글이 없습니다. <code>categories: spring</code> 으로 글을 작성하면 여기에 모입니다.</p>
{% endif %}
