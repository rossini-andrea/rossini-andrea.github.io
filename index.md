---
title: Andrea Rossini
layout: default
---

# Index

{% for post in site.posts %}
## [{{ post.title }}]({{ post.url }})

{{ post.snippet }}
{% endfor %}
