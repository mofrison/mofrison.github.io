---
title: Статьи
layout: default
category: main-menu
position: 2
lang: Ru
---

{% assign cases = site.posts | where: "categories", 'cases' %}
{% include projects.html projects = cases %}
