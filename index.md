---
layout: default
---

# {{ site.title }}

*{{ site.description }}*

---

## Latest posts

{% for post in site.posts %}
  <article>
    <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
    <p><small>{{ post.date | date: "%Y-%m-%d" }}</small></p>
    <div>
      {{ post.content }}
    </div>
    <hr>
  </article>
{% endfor %}
