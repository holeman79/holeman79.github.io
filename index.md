---
layout: default
---

# Posts

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url | relative_url }})
<small>{{ post.date | date: "%Y-%m-%d" }} · {{ post.categories | join: ", " }}</small>

{{ post.excerpt }}

---
{% endfor %}
