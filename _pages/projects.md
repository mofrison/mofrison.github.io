---
layout: default
category: main-menu
position: 1
lang: En
---

## My studies projects
These are some of my first works. They don't have any special value, but I kept them as a keepsake.
{% assign studies = site.projects | where: "tags", 'studies' %}
{% include projects.html projects = studies %}

