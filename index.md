---
title: Andrea Rossini - Homepage
layout: default
---

## Blog index

{% for post in site.posts %}
## [{{ post.title }}]({{ post.url }})

{{ post.snippet }}

*Posted on {{ page.date | date: "%-d %B %Y" }}.*

{% endfor %}
