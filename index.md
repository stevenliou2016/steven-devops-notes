---
layout: home
title: Steven DevOps Notes
---

DevOps / Backend Engineering Experiments

This site documents engineering experiments, debugging processes,
and infrastructure observations.

## Topics

- Go backend engineering
- HTTP server internals
- DevOps experiments
- Production failure analysis

---

## Latest Articles

{% for post in site.posts %}
- [{{ post.title }}]({{ site.baseurl }}{{ post.url }})
{% endfor %}
