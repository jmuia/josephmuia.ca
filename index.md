---
layout: default
---

{% for post in site.posts %}
<small class="idx-post">`{{ post.date | date: "%b %d %Y" }}`</small> [{{ post.title }}]({{ post.url }})
{% endfor %}