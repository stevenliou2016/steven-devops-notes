---
layout: home
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
- [{{ post.title }}]({{ post.url }})
{% endfor %}
