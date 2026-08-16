---
layout: default
title: Writing
permalink: /writing/
---

# Writing

Measurement notes and teardowns. Irregular — a piece goes up when a run
turns up something worth writing down, not on a schedule.

{% if site.posts.size > 0 %}
<ul class="postlist">
{% for post in site.posts %}
  <li>
    <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%-d %B %Y" }}</time>
    <a href="{{ post.url }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>
{% else %}
<p>Nothing published yet. <a href="/feed.xml">RSS</a>.</p>
{% endif %}
