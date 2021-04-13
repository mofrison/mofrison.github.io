---
title: Статьи
layout: default
category: main-menu
position: 2
lang: Ru
---

## {{page.title}}
Ниже приведены некоторые решения, реализованные мною в проектах над которыми я работал.
{% assign cases = site.posts | where: "categories", 'cases' %}
{% include projects.html projects = cases %}
