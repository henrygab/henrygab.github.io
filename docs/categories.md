---
layout: default
title: Categories
---

{% for category in site.categories %}
  <h3>{{ category[0] }} - GH Pages reversed</h3>
  <ul>
    {% for post in category[1] reversed %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
  <h3>{{ category[0] }} - sorted by date</h3>
  {% assign sorted_posts = category[1] | sort: "date" %}
  <ul>
    {% for post in category[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
  <h3>{{ category[0] }} - piped to reverse</h3>
  {% assign sorted_posts = category[1] | sort: "date" | reverse %}
  <ul>
    {% for post in category[1] | reverse %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}
