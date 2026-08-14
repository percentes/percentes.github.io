---
layout: default
title: Writing
permalink: /writing/
---

# Writing

Measurement notes and teardowns. Every piece that publishes a number
links the exact instrument commit and configuration that produced it.

<ul class="postlist">
{% for post in site.posts %}
  <li>
    <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%-d %B %Y" }}</time>
    <a href="{{ post.url }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>
