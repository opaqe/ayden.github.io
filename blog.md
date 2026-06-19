---
layout: page
title: 블로그
permalink: /blog/
---

<ul class="post-list">
  {% for post in site.posts %}
  {% unless post.categories contains 'spring' %}
  <li>
    <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
    <h3 style="margin:.2em 0;">
      <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
    </h3>
    {% if post.description %}<p>{{ post.description | escape }}</p>{% endif %}
  </li>
  {% endunless %}
  {% endfor %}
</ul>
