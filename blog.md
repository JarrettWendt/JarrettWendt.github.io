---
layout: page
title: "Blog"
permalink: /blog/
---

{% for post in site.posts %}
<h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
{{ post.excerpt }}
{% endfor %}
