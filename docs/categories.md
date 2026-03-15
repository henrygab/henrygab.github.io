---
layout: default
title: Categories
---

{% for category in site.categories %}
  <h3>{{ category[0] }}</h3>
  {% assign sorted_posts = category[1] | sort: "date" | reverse %}
  <ul>
    {% for post in category[1] | reverse %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}
