
{% for category in site.categories %}
  <h3>{{ category[0] }}</h3>
  <ul>
    {% for post in category[1] reversed %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}


Opinions expressed herein may have
(at one time) been those of the site
owner, and may not reflect those
of their employer, family, social
circle, etc.
