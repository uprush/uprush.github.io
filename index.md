---
layout: page
title: Yifeng's notes on big data, hadoop, cloud and some hobbies.
---
{% include JB/setup %}

Blog Posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
