---
title: Andrea Rossini
layout: default
---

# Andrea Rossini Blog

{% for post in site.posts %}
## [{{ post.title }}]({{ post.url }})

{{ post.snippet }}
{% endfor %}
