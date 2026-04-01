---
layout: default
title: AK's notes
author: Anthony Klein
description: AK's tech space to jot notes
---

👋 Hey.

This blog is hosted with <a href="https://github.com/KDN-Cloud/b.aklein.me" target="_blank">GitHub Pages</a> and uses Jekyll, a popular static site generator.

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) — <small>{{ post.date | date: "%b %d, %Y" }}</small>
{% endfor %}

