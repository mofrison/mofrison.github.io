---
layout: default
category: main-menu
position: 2
lang: En
---

## {{page.title}}
Below are some cases that I have implemented in projects that I have worked on.
{% assign cases = site.posts | where: "categories", 'cases' %}
{% include projects.html projects = cases %}
